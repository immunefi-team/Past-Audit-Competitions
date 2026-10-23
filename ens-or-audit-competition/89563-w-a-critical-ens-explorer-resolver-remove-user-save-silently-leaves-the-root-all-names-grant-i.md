# 89563 w a critical ens explorer resolver remove user save silently leaves the root all names grant intact so a removed manager keeps write access to every name

**Submitted on Aug 19th 2026 at 21:35:30 UTC by @Cryptor for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89563
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

A PermissionedResolver assigns write roles (`SetAddr`, `SetText`, …) per *resource*. The `ROOT_RESOURCE` (`bytes32(0)`) grants a role over **all names** the resolver serves; a per-name resource (`keccak256(namehash(name), 0)`) grants it over a single name. These are independent on-chain EACL entries.

The Explorer's resolver-roles admin UI lets an admin remove an account from a resolver, and the confirmation states it will remove the user "from all roles." But when the account holds **both** a root-scoped grant and a name-scoped grant, the removal — and the per-account Save — builds a revoke transaction scoped only to the *name*, never to the root. The `ROOT_RESOURCE` grant survives that first removal, so the account keeps write access to every name on the resolver even though the confirmation reported it removed from all roles. With that retained access the removed party overwrites any name's forward `addr(60)` record, redirecting the name's `name → address` resolution to the attacker so payments to it go to the attacker.

## Vulnerability Details

**1. Root and per-name resolver roles are independent entries; the root grant is all-names.**

`apps/portal/src/lib/roles/resolverRoles.ts`, lines 129–141:

```ts
const ROOT_RESOURCE = '0x0000…0000' // bytes32(0)
// Mirrors PermissionedResolverLib.resource(node, 0) in Solidity
export const computeNameResource = (name: string): string =>
  keccak256(encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]))
```

`computeNameResource(name)` is a keccak256 of 64 bytes and is therefore never `bytes32(0)`, so a name-scoped revoke can never touch `ROOT_RESOURCE`. Granting a user at node `''` maps to the root resource; `ResolverAddUserSheet.tsx:38-39` documents this as "Empty string = root resource (roles apply to all names)."

**2. The UI folds an account's root and name grants into one row.**

`resolverRoles.ts:179-221` `groupRolesByAccount` groups by `account` and unions every resource the account holds: `resolvedNames` is a `Set` and `decodedRoles` is the concatenation. A holder of a root grant and an `alice.eth` grant becomes a single row with `resolvedNames = ['(root)', 'alice.eth']`.

**3. Remove and Save discard the root resource.**

`ResolverRolesSidebar.tsx:169`:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

For a root+name account, `roleName` is the name (`'alice.eth'`), discarding `'(root)'`. `handleRemoveUser` (`:307-323`) sets `name: roleName` and `roles: decodedRoles` (the union of both resources' roles); `removeUserMutation` (`:266`) then calls `revokeResolverRoles({ name, account, roles, … })`. The per-account Save path `handleSaveChanges` (`:295-305`) uses the same `roleName`.

`revokeResolverRoles.ts:45-61` selects the scope from `name`:

```ts
const writeParams = revokeResolverRolesWriteParameters(walletClient,
  name === ''
    ? { resolverAddress, targetAccount: account, scope: 'root', roles }
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles })
```

With `name === 'alice.eth'`, this emits a single `scope: 'name'` revoke against `alice.eth`'s resource. **No `scope: 'root'` revoke is ever produced**, so the account's `ROOT_RESOURCE` grant is untouched — it retains `SetAddr` / `SetText` on every name the resolver serves.

**4. The root grant is the default, making the vulnerable state easy to reach.**

`ResolverAddUserSheet.tsx:39` defines `ROOT_NODE_VALUE = ''`; `:127` initializes the node selector to it (`useState<string>(ROOT_NODE_VALUE)`), shown as "Root (all nodes)" (`:253-258`); `:205` submits `name: selectedNode`. So adding a user grants a root (all-names) role by default. `grantResolverRoles.ts:47-57` mirrors the same `name === ''` scope logic. The vulnerable root+name state arises whenever such an account is also given a name-scoped grant — and the sidebar labels the resulting row "Global roles (all names)."

## Impact

The retained `ROOT_RESOURCE` role lets the removed party write the **forward** `addr(60)` record — the `name → address` mapping used to pay a name — of every name the resolver serves. After the incomplete removal, the party calls `setAddr(namehash(victimName), 60, attackerAddress)`, and from then on the app's own resolution (`getResolvedAddress` → the Explorer's address view and its send/transfer recipient fields) and any external wallet resolve `victimName` **forward** to the *attacker's* address. A user who pays `victimName` — the ordinary way an ENS name is used — sends their funds to the attacker. This is the program's "resolving or rendering an ENS name to an attacker-controlled address," realized on the forward payment path rather than a reverse-lookup display: the record overwritten is the exact `addr(60)` value the application reads to decide where a payment goes.

The attacker is a party the administrator explicitly removed. The dialog confirms the removal is complete ("remove this user from all roles"), yet only the name-scoped grant is revoked; the all-names `ROOT_RESOURCE` grant remains and the app gives the admin no indication the removed party still controls resolution. That control extends to **every** name the resolver serves — including names owned by other users of a shared resolver — and persists until an administrator independently notices the account still present in the roles list and removes it a second time (a second "Remove user" on the now root-only row does emit a `scope: 'root'` revoke). The per-account Save path has the same defect — un-checking a permission on a root+name account only touches the name resource, leaving the equivalent all-names permission in force. Preconditions: the removed account held both a root and a name grant on the resolver (root is the default grant when adding a user), and the admin relied on the Remove/Save confirmation.

## Proof of Concept

## Proof of Concept

### Setup (per the program Resources page)

Requirements: Node **v22**, pnpm **v10**. Audited revision: commit `63772fd872af472ced58b009499355f3430c2a86` (`immunefi-team/audit-comp-ens`, fork commit `cda79ac`).

```bash
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens
git checkout 63772fd872af472ced58b009499355f3430c2a86   # or its fork commit cda79ac
corepack enable
pnpm install --frozen-lockfile
```

### A. Coded PoC (deterministic; drives the real revoke builder)

Create the file `apps/portal/src/features/resolver/helpers/revokeResolverRoles.poc.test.ts` with exactly this content:

```ts
import type { Address, WalletClient } from 'viem'
import { describe, expect, it, vi } from 'vitest'
import {
  buildResourceToNameMap, computeNameResource, groupRolesByAccount, resolverRoles,
} from '@/lib/roles/resolverRoles'

const capturedRevokes: Array<{ scope?: string; name?: string }> = []
vi.mock('@ensdomains/ensjs/wallet/v2', () => ({
  revokeResolverRolesWriteParameters: (_w: unknown, p: { scope?: string; name?: string }) => {
    capturedRevokes.push({ scope: p.scope, name: p.name })
    return { abi: [{ type: 'function', name: 'revokeRoles', stateMutability: 'nonpayable', inputs: [], outputs: [] }], functionName: 'revokeRoles', args: [] }
  },
}))
vi.mock('@ens-apps/transaction-manager', () => ({ transactionManager: { startTransaction: vi.fn(() => 'tx') }, waitForTransaction: vi.fn() }))
vi.mock('@/features/transaction-manager/helpers/intents', () => ({ toEoaCustomIntent: (i: unknown) => i }))
import { prepareRevokeResolverRolesTransaction } from './revokeResolverRoles'

const ROOT_RESOURCE = '0x' + '0'.repeat(64)
const MAL = '0x00000000000000000000000000000000000000ff' as Address
const walletClient = { account: { address: '0x0000000000000000000000000000000000009999' as Address }, chain: { id: 11155111 } } as unknown as WalletClient
const removeUser = (name: string) => {
  capturedRevokes.length = 0
  prepareRevokeResolverRolesTransaction({ resolverAddress: '0x0000000000000000000000000000000000001111' as Address, name, account: MAL, roles: ['ROLE_SET_ADDR'] as never, walletClient, chainId: 11155111 })
}

describe('R-2: resolver "Remove user" leaves the root (all-names) grant intact', () => {
  it('a root grant + a name grant union into one UI row', () => {
    const [group] = groupRolesByAccount(
      [{ account: MAL, resource: ROOT_RESOURCE, roleBitmap: resolverRoles.ROLE_SET_ADDR },
       { account: MAL, resource: computeNameResource('alice.eth'), roleBitmap: resolverRoles.ROLE_SET_ADDR }] as never,
      buildResourceToNameMap([{ name: 'alice.eth' }]))
    expect(group.resolvedNames).toEqual(expect.arrayContaining(['(root)', 'alice.eth']))
  })
  it('the removal discards "(root)" and revokes ONLY the name scope -> the root grant survives', () => {
    const roleName = ['(root)', 'alice.eth'].find((n) => n !== '(root)') ?? '' // ResolverRolesSidebar.tsx:169
    expect(roleName).toBe('alice.eth')                                          // '(root)' discarded
    removeUser(roleName)
    expect(capturedRevokes).toEqual([{ scope: 'name', name: 'alice.eth' }])     // never scope:'root'
    expect(capturedRevokes.some((r) => r.scope === 'root')).toBe(false)
  })
  it('control: a root-only account (roleName === "") correctly revokes root', () => {
    removeUser(['(root)'].find((n) => n !== '(root)') ?? '')
    expect(capturedRevokes[0]?.scope).toBe('root')
  })
})
```

Run:

```
pnpm --filter portal exec vitest run src/features/resolver/helpers/revokeResolverRoles.poc.test.ts
```

Result (all pass): for a root+name account the removal captures exactly `{ scope: 'name', name: 'alice.eth' }` and never a `scope: 'root'` revoke — proving the on-chain `ROOT_RESOURCE` grant is left in place. The control shows the builder *can* revoke root (only when the account is root-only), confirming the defect is the UI discarding `'(root)'`. This is the in-scope client-side defect; Section B carries it through to the demonstrated fund-misdirection — the removed party uses the retained root role to overwrite a name's forward `addr(60)`, and the app then resolves that name to the attacker's address.

### B. End-to-end reproduction (browser + Sepolia)

Prerequisites: a PermissionedResolver you administer on Sepolia (an account with `canManageRoles`), the running Explorer (`pnpm --filter portal dev`, `http://localhost:3001`), and an attacker account `0xMAL`.

1. In the resolver's roles view, click **Add user**, enter `0xMAL`, leave the node selector on its **default "Root (all nodes)"**, select `SetAddr` / `SetText`, and submit. `0xMAL` now holds a `ROOT_RESOURCE` grant (all names).
2. Add `0xMAL` again, this time choosing a specific name node (for example `alice.eth`), and submit. `0xMAL` now also holds a name-scoped grant.
3. The roles table shows a single `0xMAL` row labelled "Global roles (all names)".
4. Open that row and click **Remove user**, then confirm.
5. Observe the single submitted transaction: `revokeRoles` scoped to `alice.eth`'s resource only — no root revoke. The UI reports the user removed.
6. As `0xMAL` (the "removed" account), call `setAddr(namehash('bob.eth'), 60, 0xMAL)` on the resolver — `bob.eth` is a name the resolver serves that `0xMAL` no longer administers. It succeeds, because `0xMAL` still holds the `ROOT_RESOURCE` role. This overwrites `bob.eth`'s **forward `addr(60)`** record.
7. In the Explorer, resolve `bob.eth` **forward** (name → address): open its address/profile view, or paste `bob.eth` as the recipient in a send/transfer flow. The app now returns **`0xMAL`** (the attacker) as `bob.eth`'s address. A user who pays `bob.eth` is therefore sent to the attacker's address — a payment misdirected by a party the administrator had already removed. This is the demonstrated fund-misdirection: the value the application resolves and would send to was overwritten by the retained root role.
8. (Recoverability, for completeness) The `0xMAL` row reappears in the roles list as a root-only entry; a second "Remove user" on it emits the `scope: 'root'` revoke. Full removal therefore takes two actions, and the first — the one the confirmation says removed the user "from all roles" — leaves the attacker in control of resolution in the interim.
