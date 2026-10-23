# 89418 w a critical broken access control in the resolver remove user flow leaves a revoked delegate able to redirect the address records of every name on the resolver

**Submitted on Aug 19th 2026 at 07:20:52 UTC by @FaithfulLayney for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89418
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer's resolver-roles page groups a delegate's roles from every resource of a `PermissionedResolver` into one table row, but "Remove user" and "Save" submit a single revocation scoped to one resource. Role bits held at any other resource are left in place, since de-authorizing a bit an account does not hold at that resource is a no-op. An administrator removing a compromised or departed delegate gets a successful transaction and a "removed from all roles" confirmation while the delegate keeps root-scoped `ROLE_SET_ADDR`, which lets it rewrite the ETH address record of every name that resolver serves. Clients then resolve those names to an attacker-controlled address and users paying them send funds to the attacker.

## Vulnerability Details

A `PermissionedResolver` scopes roles per resource: the root resource grants authority over all names, and each name has its own resource. Both scopes appear in the same roles table.

`groupRolesByAccount` keys on the account address alone and unions the decoded role bits across every resource that account appears in, keeping only the resource labels:

`apps/portal/src/lib/roles/resolverRoles.ts:193-214`

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
    ...
  }
}
```

An account holding `ROLE_SET_ADDR` at root and `ROLE_SET_TEXT` at `name1.eth` yields one row with `decodedRoles = {ROLE_SET_ADDR, ROLE_SET_TEXT}` and `resolvedNames = ['(root)', 'name1.eth']`. The bit-to-resource mapping is lost here and is not recovered downstream.

The sidebar reduces the row to a single revocation target, the first non-root name:

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:166-169`

```ts
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`roleName` is `''`, the value that selects root scope, only when the account holds root roles and nothing else. Once it holds any name-scoped role, `roleName` is that name.

Both mutating handlers pass that single `roleName` with the unioned `decodedRoles`:

`ResolverRolesSidebar.tsx:307-323`

```ts
const handleRemoveUser = () => {
  if (!selectedAccount || decodedRoles.length === 0 || removeUserMutation.isPending) return

  removeUserMutation.reset()
  setPendingAction({
    type: 'remove',
    name: roleName,
    account: selectedAccount,
    roles: decodedRoles,
  })
  openModal()
}
```

`handleSaveChanges` (`:295-305`) passes the same `name: roleName` for per-permission edits.

The scope is resolved in the call builder, which emits one call:

`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61`

```ts
const writeParams = revokeResolverRolesWriteParameters(
  walletClient,
  name === ''
    ? { resolverAddress, targetAccount: account, scope: 'root',  roles }
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles },
)
```

In the pinned `@ensdomains/ensjs`, `scope: 'root'` encodes `revokeRootRoles(bitmap, account)` and `scope: 'name'` encodes `authorizeNameRoles(dnsEncode(name), bitmap, account, false)`. Because `roleName` is a name, only the name-scoped call is ever produced, and `revokeRootRoles` is never emitted for an account that holds any name-scoped role. De-authorizing `ROLE_SET_ADDR` at `name1.eth` when the account holds that bit at root changes no state.

The same expression governs later attempts, so the residual root roles cannot be targeted from this page at all; clearing them requires calling `revokeRootRoles` on the resolver directly.

The UI reports the opposite. The row header prints "Global roles (all names)" when the account holds root roles (`:393-396`), the confirmation dialog reads "Are you sure you want to remove this user from all roles? This action cannot be undone." (`:445-447`), and the transaction is labelled `Remove user <account>` (`:343`).

The multi-resource row is the default result of the app's own grant flow. The Add-user sheet initialises its scope selector to the root value, so a delegate added without opening the node combobox receives root-scoped roles:

`apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:39,127`

```ts
const ROOT_NODE_VALUE = ''
...
const [selectedNode, setSelectedNode] = useState<string>(ROOT_NODE_VALUE)
```

## Attack path

{% stepper %}
{% step %}

### Grant a root-scoped role

Alice adds Bob as a manager on her resolver through the Add-user sheet, leaving the scope selector at its default, and grants Set Address. Bob holds `ROLE_SET_ADDR` at the root resource, covering every name the resolver serves.
{% endstep %}

{% step %}

### Grant a name-scoped role

Bob is later granted a role scoped to one name, `name1.eth`. His row now reads "Global roles (all names)".
{% endstep %}

{% step %}

### Remove the user through the application

Alice opens the row, clicks Remove user, confirms "remove this user from all roles", and signs.
{% endstep %}

{% step %}

### The application submits a name-scoped revocation

The app submits `authorizeNameRoles(dnsEncode("name1.eth"), <union bitmap>, Bob, false)`. Bob never held `ROLE_SET_ADDR` at `name1.eth`, so that bit is a no-op; `revokeRootRoles` is not submitted. Bob keeps `ROLE_SET_ADDR` at root.
{% endstep %}

{% step %}

### Rewrite address records

Bob rewrites the ETH address record of every name the resolver serves to an address he controls.
{% endstep %}

{% step %}

### Users send funds to the attacker

The Manager and Explorer resolve and render those names to Bob's address; users paying `alice.eth` or any name on that resolver send funds to Bob.
{% endstep %}

{% step %}

### Root roles remain unreachable in the application

Alice cannot revoke Bob from the page: `roleName` still resolves to `name1.eth`, so the root scope is never targeted again.
{% endstep %}
{% endstepper %}

## Impact Details

Selected impact (Critical, Websites & Apps): "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: — resolving or rendering an ENS name to an attacker-controlled address — showing the wrong recipient or amount on a transaction the user then approves."

The residual role is `ROLE_SET_ADDR` on the root resource, so the exposure covers every name the affected resolver serves rather than a single record.

Losses:

* All inbound payments to every name on that resolver. An address record exists to receive funds; once rewritten, each subsequent payment goes to the attacker. The amount is not bounded by any balance the application holds — it is the full flow of payments to those names for as long as the record stands. For a resolver serving a business's primary name and its subnames, that is all of its ENS-addressed receipts.
* Duration. The administrator receives no in-app signal that the removal failed, and the page cannot target the residual root roles afterwards. Detection requires reading role state on-chain independently; remediation requires a direct `revokeRootRoles` call outside the product. Payments continue to reach the attacker until then.
* Any other root-scoped bits the account held survive the same way, including `ROLE_SET_NAME`, which rewrites the reverse/primary-name record so the attacker's address displays under the victim's name, and `ROLE_UPGRADE`, which replaces the resolver implementation and so covers every record type at once.

The loss falls on the name owner and on third parties paying those names. The attacker is a delegate whose authority the owner revoked through the product's own control and was told had been removed; at that point the delegate holds no authority the owner intended it to keep. The administrator side of the flow needs no role beyond resolver ownership and no interaction beyond the documented way to remove a delegate.

## Recommendation

Revoke per resource. The account's group already carries every resource it appears in, so iterate them and emit one revocation per resource with the bits held there:

```diff
-const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
+// One revoke per resource the account holds roles on; '(root)' maps to the
+// root-scoped revoke (empty name).
+const revokeTargets = resolvedNames.map((n) => (n === '(root)' ? '' : n))
```

Submit one `revokeResolverRoles({ name, roles })` per target, taking `roles` from the per-resource `roleBitmap` in `row.original.roles` rather than the flattened `decodedRoles`. Apply the same loop to `handleSaveChanges`, which shares `roleName`. A smaller interim fix is to always add a root-scoped revoke when `resolvedNames` contains `'(root)'`.

While the revoke stays single-resource, the confirmation dialog and row header should name the resource being changed instead of "all roles" and "Global roles (all names)".

## References

* `groupRolesByAccount`, role bit definitions and `computeNameResource`: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L193-L214> <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L24-L45>
* `ResolverRolesSidebar` — target selection, remove and save handlers, UI copy: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L166-L169> <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L295-L323> <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L389-L397> <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L449>
* `prepareRevokeResolverRolesTransaction` — root vs name scope: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L75>
* `ResolverAddUserSheet` — grant scope defaults to root: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L39> <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L127>
* `@ensdomains/ensjs` `revokeResolverRolesWriteParameters` — `scope: 'root'` → `revokeRootRoles`, `scope: 'name'` → `authorizeNameRoles(..., false)`, in `dist/actions/wallet/v2/revokeResolverRoles.js` of the pinned dependency

## Proof of Concept

### Step-by-step (in the running application)

Preconditions: you own a `PermissionedResolver` in the Explorer and can manage its roles.

{% stepper %}
{% step %}

### Grant Bob a root-scoped role

Open the resolver's **Roles** page and click **Add user**. Enter a delegate address (Bob), tick **Set Address**, and submit **without opening the node combobox** — it defaults to *Root (all nodes)*. Bob now holds `ROLE_SET_ADDR` on the **root** resource, i.e. over every name this resolver serves.
{% endstep %}

{% step %}

### Grant Bob a name-scoped role

Add Bob a second, name-scoped role: **Add user** again for the same address, this time selecting one name (`name1.eth`) and ticking e.g. **Set Text**. The roles table now shows Bob in a single row labelled **"Global roles (all names)"**.
{% endstep %}

{% step %}

### Remove Bob through the UI

Click Bob's row to open the sidebar, then **Remove user**, and confirm the dialog: *"Are you sure you want to remove this user from all roles? This action cannot be undone."* Sign the transaction.
{% endstep %}

{% step %}

### Observe the apparent success

Observe the application reports success and Bob disappears from the table.
{% endstep %}

{% step %}

### Inspect the submitted transaction

Inspect the submitted transaction: it is `authorizeNameRoles(dnsEncode("name1.eth"), <bitmap>, Bob, false)` — a **name-scoped** de-authorization. No `revokeRootRoles` transaction was submitted.
{% endstep %}

{% step %}

### Verify the root role remains

Read the resolver state directly: `hasRoles(ROOT_RESOURCE, ROLE_SET_ADDR, Bob)` is still **true**. Bob can call the resolver and rewrite the ETH address record of every name it serves; the Manager and Explorer then resolve those names to Bob's address.
{% endstep %}

{% step %}

### Attempt to remove root roles again

Re-open the roles page and try to remove Bob's root roles again — the flow always targets `name1.eth`, so the root-scoped roles cannot be revoked through the application at all.
{% endstep %}
{% endstepper %}

### Automated proof against the application's own code

The test drives the real code path — `groupRolesByAccount`, the target-selection expression from `ResolverRolesSidebar` (`roleName` / `decodedRoles`), and `prepareRevokeResolverRolesTransaction` — and decodes the calldata that would be signed.

Save as `apps/portal/src/features/resolver/helpers/poc-revoke-underscoped.test.ts` in a clean clone of `immunefi-team/audit-comp-ens` at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, then run:

```bash
pnpm install
pnpm --filter @ens-apps/portal exec vitest run src/features/resolver/helpers/poc-revoke-underscoped.test.ts
```

```ts
/**
 * PoC — resolver "Remove user" revoke is silently scoped to ONE resource.
 *
 * Uses the app's OWN code: `groupRolesByAccount` (the roles-table aggregation),
 * the exact `ResolverRolesSidebar` target-selection logic (`roleName`/`decodedRoles`),
 * and `prepareRevokeResolverRolesTransaction` (the real revoke-call builder).
 *
 * Scenario: delegate `BOB` holds ROLE_SET_ADDR + ROLE_SET_NAME at ROOT (all names) and
 * ROLE_SET_TEXT at `name1.eth`. The admin clicks "Remove user". The app builds a
 * SINGLE revoke scoped to `name1.eth` — so BOB's ROOT ROLE_SET_ADDR (the addr-record /
 * fund-redirect primitive) is never revoked, and the ROOT scope is never even targeted.
 */
import {
  permissionedResolverAuthorizeNameRolesSnippet,
  permissionedResolverRevokeRootRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import {
  createWalletClient,
  decodeFunctionData,
  http,
  slice,
  toFunctionSelector,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { describe, expect, it } from 'vitest'
import {
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from '@/lib/roles/resolverRoles'
import { prepareRevokeResolverRolesTransaction } from './revokeResolverRoles'

const asAbi = (s: unknown) => (Array.isArray(s) ? s : [s]) as never
// ensjs maps revoke scope -> function: 'root' => revokeRootRoles, 'name' => authorizeNameRoles(...,false)
const ROOT_REVOKE_SELECTOR = toFunctionSelector(
  asAbi(permissionedResolverRevokeRootRolesSnippet)[0],
)
const NAME_DEAUTH_SELECTOR = toFunctionSelector(
  asAbi(permissionedResolverAuthorizeNameRolesSnippet)[0],
)

const ROOT_RESOURCE =
  '0x0000000000000000000000000000000000000000000000000000000000000000'
const BOB = '0x00000000000000000000000000000000000000b0' as const
const RESOLVER = '0x000000000000000000000000000000000000dead' as const

// Bob's on-chain roles: ROOT (all names) SET_ADDR+SET_NAME, and name1.eth SET_TEXT.
const NAME1 = 'name1.eth'
const roleRows = [
  {
    account: BOB,
    resource: ROOT_RESOURCE,
    roleBitmap: (resolverRoles.ROLE_SET_ADDR | resolverRoles.ROLE_SET_NAME).toString(),
  },
  {
    account: BOB,
    resource: computeNameResource(NAME1),
    roleBitmap: resolverRoles.ROLE_SET_TEXT.toString(),
  },
]
const resourceToName = new Map<string, string>([
  [ROOT_RESOURCE, '(root)'],
  [computeNameResource(NAME1).toLowerCase(), NAME1],
])

describe('resolver revoke under-scoped to one resource', () => {
  it('aggregates Bob roles across ROOT + name1 into one row (real groupRolesByAccount)', () => {
    const [row] = groupRolesByAccount(roleRows, resourceToName)
    expect(row.account).toBe(BOB.toLowerCase())
    // union across resources: the row claims Bob holds SET_ADDR (only at root) AND SET_TEXT (only at name1)
    expect(row.decodedRoles).toContain('ROLE_SET_ADDR')
    expect(row.decodedRoles).toContain('ROLE_SET_NAME')
    expect(row.decodedRoles).toContain('ROLE_SET_TEXT')
    expect([...row.resolvedNames].sort()).toEqual(['(root)', NAME1])
  })

  it('"Remove user" builds a revoke scoped to name1, NEVER touching ROOT — Bob keeps root SET_ADDR', () => {
    const [row] = groupRolesByAccount(roleRows, resourceToName)

    // --- exact ResolverRolesSidebar.tsx logic ---
    const decodedRoles = row.decodedRoles // union bitmap set
    const resolvedNames = row.resolvedNames
    const roleName = resolvedNames.find((n) => n !== '(root)') ?? '' // line 169
    // handleRemoveUser -> revokeResolverRoles({ name: roleName, roles: decodedRoles })

    // The defect in one assertion: despite Bob holding ROOT roles, the target is a NAME, not root.
    expect(roleName).toBe(NAME1)
    expect(roleName).not.toBe('') // '' would mean scope:'root'

    // Build the real revoke intent the app would submit.
    const walletClient = createWalletClient({
      account: privateKeyToAccount(`0x${'11'.repeat(32)}`),
      chain: sepolia,
      transport: http(),
    })
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: BOB,
      roles: decodedRoles as never,
      walletClient,
      chainId: sepolia.id,
    })

    const data = (intent.request as { data: `0x${string}` }).data
    const selector = slice(data, 0, 4)

    // The single produced call is a NAME-scoped de-authorization (authorizeNameRoles(...,false)),
    // NOT a root revoke (revokeRootRoles). So no root-scoped role of Bob is ever revoked.
    expect(selector).toBe(NAME_DEAUTH_SELECTOR)
    expect(selector).not.toBe(ROOT_REVOKE_SELECTOR)

    // Decode the name-scoped call and confirm it targets name1 + Bob with authorized=false.
    const decoded = decodeFunctionData({
      abi: asAbi(permissionedResolverAuthorizeNameRolesSnippet),
      data,
    }) as unknown as { functionName: string; args: readonly unknown[] }
    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect((decoded.args.at(-2) as string).toLowerCase()).toBe(BOB.toLowerCase())
    expect(decoded.args.at(-1)).toBe(false) // authorized=false → de-authorize, at name1 only

    // Corollary: the correct fix would additionally emit revokeRootRoles for Bob's ROOT bits.
    // The app never does (roleName is a name, never ''), so on-chain
    //   hasRoles(ROOT_RESOURCE, ROLE_SET_ADDR, Bob) stays TRUE after "full" removal.
    expect(roleName).not.toBe('') // '' is the ONLY path that yields revokeRootRoles
  })
})
```

Output:

```
 RUN  v4.1.5 /…/audit-comp-ens/apps/portal

 ✓ src/features/resolver/helpers/poc-revoke-underscoped.test.ts > resolver revoke under-scoped to one resource > aggregates Bob roles across ROOT + name1 into one row (real groupRolesByAccount) 1ms
 ✓ src/features/resolver/helpers/poc-revoke-underscoped.test.ts > resolver revoke under-scoped to one resource > "Remove user" builds a revoke scoped to name1, NEVER touching ROOT — Bob keeps root SET_ADDR 17ms

 Test Files  1 passed (1)
      Tests  2 passed (2)
   Duration  1.62s
```

### What the run establishes

* `groupRolesByAccount` returns **one** row for Bob whose `decodedRoles` is the union `{ROLE_SET_ADDR, ROLE_SET_NAME, ROLE_SET_TEXT}` across the root and `name1.eth` resources, and whose `resolvedNames` is `['(root)', 'name1.eth']` — the row no longer records which bit came from which resource.
* The sidebar's own expression resolves `roleName` to `'name1.eth'`, never `''`, so the root branch of the scope selector is unreachable for this account.
* The single transaction the application would sign decodes to `authorizeNameRoles(dnsEncode("name1.eth"), <union bitmap>, Bob, authorized = false)`. Its selector is the name-scoped one and is **not** `revokeRootRoles`.
* Therefore no root-scoped bit is revoked. Bob's `ROLE_SET_ADDR` at the root resource survives the "removal", and de-authorizing it at `name1.eth` — where he never held it — is an on-chain no-op.
* Negative control / after-fix expectation: with a per-resource revoke, the same flow additionally emits `revokeRootRoles(<root bits>, Bob)`, and `hasRoles(ROOT_RESOURCE, ROLE_SET_ADDR, Bob)` becomes false.

No transaction was sent to any live network; the test builds and decodes calldata locally.
