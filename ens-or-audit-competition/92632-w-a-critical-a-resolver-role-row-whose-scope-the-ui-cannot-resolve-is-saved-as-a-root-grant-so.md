# 92632 w a critical a resolver role row whose scope the ui cannot resolve is saved as a root grant so one click on a single account s row signs grantrootroles over every name on the resolver

**Submitted on Sep 11th 2026 at 05:19:20 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92632
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

On the Explorer's resolver roles sidebar, a role row whose scope the UI cannot resolve is silently converted into a **ROOT-scoped grant**. Clicking "Save changes" on one account's narrow row therefore signs `grantRootRoles(...)` — authority over **every name on that resolver** — while the screen shows a single account, names no scope anywhere, and the transaction modal describes the action only as "Update roles for 0x…".

The whole defect is one `??`.

## The collapse, in four lines

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169`

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

When `resolvedNames` is `[]`, `.find()` returns `undefined` and the `??` produces `''`. The very next layer documents `''` as meaning ROOT — `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:20`:

```ts
/** Dotted name (e.g. "myname.eth") or empty string for ROOT_RESOURCE (all names). */
readonly name: string
```

`apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-60`:

```ts
  const writeParams = grantResolverRolesWriteParameters(
    walletClient,
    name === ''
      ? { resolverAddress, targetAccount: account, scope: 'root', roles }
      : { resolverAddress, targetAccount: account, scope: 'name', name, roles },
  )
```

and ensjs turns `scope: 'root'` into the resolver-wide call — `@ensdomains/ensjs/actions/wallet/v2/grantResolverRoles.ts:116-122`:

```ts
    case 'root':
      return {
        ...base,
        abi: permissionedResolverGrantRootRolesSnippet,
        functionName: 'grantRootRoles',
        args: [encodeResolverRoleBitmap(params.roles), params.targetAccount],
      } as const
```

So "I could not work out which name this row is for" and "this row is for all names" are the same value.

## Why a row resolves to nothing

`buildResourceToNameMap` only ever contains ROOT plus the **part-0** resource of each listed node — `apps/portal/src/lib/roles/resolverRoles.ts:138-141` and `:147-161`:

```ts
export const computeNameResource = (name: string): string =>
  keccak256(
    encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]),
  )
```

```ts
  const map = new Map<string, string>()
  map.set(ROOT_RESOURCE, '(root)')

  for (const node of nodes) {
    if (node.name) {
      map.set(computeNameResource(node.name).toLowerCase(), node.name)
    }
  }
```

and `groupRolesByAccount` drops anything it cannot find — `:196-206`:

```ts
    const resolvedName = resourceToName?.get(resource) ?? null
    ...
      if (resolvedName) existing.resolvedNames.add(resolvedName)
```

That leaves two reachable ways for a row to carry a resource the map does not hold:

1. **A role scoped to one part.** `PermissionedResolverLib.resource(node, part)` is `keccak256(node ‖ part)`; `computeNameResource` is only its `part = 0` case. A role scoped to a single text key or a single coin type has `part = keccak(key)` / `part = coinType`, so it is never in the map.
2. **A node the list does not currently include** — the name was re-pointed at a different resolver, the indexer is behind, or the `nodes` query is paginated. The role row survives; the node entry does not.

The app already has the correct answer and throws it away: `ResolverRole.name` is fetched from the indexer (`apps/portal/src/features/resolver/hooks/useResolverOverview.ts:21`, `:97`) and `groupRolesByAccount` never reads it.

## Nothing on screen warns

Three places could disclose the scope. In this state, none does.

`ResolverRolesSidebar.tsx:391` — the only scope label is gated on the array being non-empty, so it renders nothing at all:

```tsx
{selectedAccount && resolvedNames.length > 0 && (
  <p className="text-sm text-muted-foreground">
    {resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
  </p>
)}
```

`ResolverRolesTable.tsx:70` — the Name column renders `null`.

`ResolverRolesSidebar.tsx:348` — the transaction modal names the account only:

```ts
      transactionName: `Update roles for ${pendingAction?.account ?? ''}`,
```

The admin sees one account, one set of checkboxes, and no indication that the word "all" is involved.

## Impact

`grantRootRoles` on a PermissionedResolver hands the target account the selected roles at ROOT, i.e. for every name the resolver serves. With `ROLE_SET_ADDR` the grantee can rewrite the ETH address record of **every** name on that resolver, so anyone resolving any of those names sends funds to an address the grantee controls. `ROLE_SET_ALIAS` lets them alias any of those names to a node they own; `ROLE_SET_TEXT` / `ROLE_SET_CONTENTHASH` let them rewrite the rest of the profile.

The transaction succeeds rather than reverting: a resolver's deployer is initialized with the full admin bitmap at root (`apps/portal/src/features/resolver/utils/permissionedResolver.ts:21-23`, `:39-45` — `initialize(owner, 0x1111…1111, [])`), which is exactly the admin variant `grantRootRoles` requires. The admin doing the routine maintenance is the one who holds it.

**Attacker path.** The attacker needs one narrow role row on a shared resolver — the ordinary state for any delegate — and then needs that row to stop resolving. Either arm of the previous section does it, and the second needs nothing but re-pointing their own name's resolver, which is a one-click portal action. The next time the admin edits that row, the save is a root grant.

It also fires with no attacker at all: indexer lag or a paginated `nodes` list produces the same empty `resolvedNames`, so an admin can hand out resolver-wide authority to a legitimate delegate by accident.

## Severity

**Critical**, under "causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user … submits a transaction to an unintended recipient" — the transaction detail that is wrong is its scope, and the downstream consequence is the row's own example, "resolving or rendering an ENS name to an attacker-controlled address". It is equally "modifying transaction arguments or parameters" on an already-connected wallet: the argument that reaches the chain is a resolver-wide grant, and the UI represented a single row.

## Why this is not an already-known or already-closed issue

**Not the resolver/role revocation family.** Those findings are all *failures to remove* — a revoke that is narrower than the UI claimed, or a stale role surviving. This is the opposite direction: a **grant that silently widens** from name-scoped to resolver-wide. The axis is scope creation, not scope retention.

**Not in the published known issues.** The known-issues document contains no entry touching resolver role scope, `grantRootRoles`, or `authorizeNameRoles`. Its Part 2 preamble draws exactly this line:

> "a demonstrated *security* consequence of one of them (for example, **a role combination that grants authority it should not**, rather than merely erroring) goes beyond the known breakage and is a new finding."

**Not a normalization issue.** No homoglyph, no Unicode, no name parsing is involved. The bug is a sentinel collision in plain ASCII and reproduces with ordinary names.

## Recommended fix

Stop using `''` to mean two different things. `resolvedNames` being empty means *unknown*, which should block the save, not widen it:

* Make the scope explicit — carry the row's `resource` (or the `ResolverRole.name` the indexer already returns and `groupRolesByAccount` discards) through to the grant, rather than re-deriving a name and using `''` as a sentinel.
* Fail closed: if the scope cannot be resolved, disable "Save changes" and say why.
* Render the scope unconditionally. Removing the `resolvedNames.length > 0` guard at `ResolverRolesSidebar.tsx:391` so an unresolved row reads "Scope unknown" would, on its own, have made this visible.
* Name the scope in the transaction modal at `:348`, so a root grant cannot be approved while reading "Update roles for 0x…".

## Proof of Concept

## What this proves

The test drives the REAL `buildResourceToNameMap` and `groupRolesByAccount` from `apps/portal/src/lib/roles/resolverRoles.ts`, reproduces the sidebar's own `?? ''` expression verbatim, and feeds the result into the REAL ensjs `grantResolverRolesWriteParameters`. Nothing in the path under test is stubbed; the only stand-in is a wallet client carrying a `chain` and an `account`, which ensjs copies onto the returned params and which cannot affect `functionName` or `args`.

Four assertions:

1. **control** — a part-0 row resolves to its name, so the sidebar has a scope to show and the calldata is `authorizeNameRoles` carrying exactly that one DNS-encoded name;
2. **the escalation** — a role scoped to a single text key (`part = keccak("com.twitter")`) is absent from the map, `resolvedNames` is `[]`, the scope line's `length > 0` guard renders nothing, the sidebar expression yields `''`, and the calldata becomes `grantRootRoles(bitmap, account)`;
3. **the no-attacker variant** — a node missing from the list (re-pointed resolver, indexer lag, pagination) produces the identical collapse;
4. **blast radius** — the name-scoped call encodes one specific name; the root call encodes none, so it cannot exclude any of the resolver's names.

## Environment

Frozen revision under test: 1c9b47f. Nothing is deployed and no live service is contacted.

```
pnpm install
```

Save the file below as:

```
apps/portal/src/features/resolver/poc-resolver-role-scope-escalation.test.ts
```

Run it:

```
cd apps/portal
pnpm vitest run src/features/resolver/poc-resolver-role-scope-escalation.test.ts
```

## Result

```
RUN  v4.1.5 .../apps/portal

  [control]   resolvedNames=["victim.eth"] -> authorizeNameRoles(victim.eth)
  [ESCALATED] resolvedNames=[] -> grantRootRoles(bitmap, attacker) — ALL names
  [ESCALATED] unlisted node -> grantRootRoles — no attacker action needed beyond re-pointing
  [blast]     authorizeNameRoles targets 1 name; grantRootRoles targets all 3

 Test Files  1 passed (1)
      Tests  4 passed (4)
```

## Reproducing in the UI

1. Deploy a PermissionedResolver from the Explorer (`/$name/registry` → Configure registry) and point two or more names at it, so the roles table has more than one name to be resolver-wide over.
2. Grant a delegate a role on one of those names.
3. Put that delegate's row into the unresolved state, either by granting the role at a non-zero part (a text-key- or coin-scoped role) or by re-pointing that name's resolver elsewhere so the indexer stops listing the node.
4. Open `/resolver/$address/roles`, select the delegate, and observe: no scope line, an empty Name column, and a modal reading only `Update roles for 0x…`.
5. Toggle any permission and click **Save changes**. The signed call is `grantRootRoles`, not `authorizeNameRoles`.

## Verifying the call sites

```
grep -n "resolvedNames.find" apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx
grep -n "resolvedNames.length > 0" apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx
grep -n "transactionName" apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx
grep -n "name === ''" -A 12 apps/portal/src/features/resolver/helpers/grantResolverRoles.ts
grep -n "computeNameResource|ROOT_RESOURCE" -A 6 apps/portal/src/lib/roles/resolverRoles.ts
grep -n "resourceToName?.get" apps/portal/src/lib/roles/resolverRoles.ts
```

## Full test file

/\*\*

* PoC — a resolver role row whose scope the UI cannot resolve is silently
* escalated to ROOT scope, so "save" on one account's narrow row ships
* `grantRootRoles(...)` — authority over EVERY name on that resolver.
*
* The collapse is one `??`:
*
* features/resolver/components/ResolverRolesSidebar.tsx:169
* ```
  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
  ```
*
* `resolvedNames` is `[]` whenever the row's `resource` is not in the
* resource->name map (see below). `.find()` then returns `undefined` and the
* `??` turns "scope unknown" into `''` — which the very next layer documents
* as meaning ROOT:
*
* features/resolver/helpers/grantResolverRoles.ts:20
* ```
  /** Dotted name (e.g. "myname.eth") or empty string for ROOT_RESOURCE (all names). *\/
  ```
* ```
  readonly name: string
  ```
*
* features/resolver/helpers/grantResolverRoles.ts:47
* ```
  name === ''
  ```
* ```
    ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  ```
* ```
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
  ```
*
* and ensjs turns `scope: 'root'` into the resolver-wide grant:
*
* @ensdomains/ensjs/actions/wallet/v2/grantResolverRoles.ts:116-122
* ```
  case 'root': functionName: 'grantRootRoles',
  ```
* ```
               args: [encodeResolverRoleBitmap(roles), targetAccount]
  ```
*
* Nothing on screen says "all names" in this state. The sidebar's only scope
* label is gated on the array being non-empty, so it renders NOTHING:
*
* ResolverRolesSidebar.tsx:391
* ```
  {selectedAccount && resolvedNames.length > 0 && (
  ```
* ```
    <p>{resolvedNames.includes('(root)') ? 'Global roles (all names)' : ...}</p>
  ```
* ```
  )}
  ```
*
* and the transaction modal names only the account, never the scope:
*
* ResolverRolesSidebar.tsx:348
* ```
  transactionName: `Update roles for ${pendingAction?.account ?? ''}`
  ```
*
* WHY A ROW RESOLVES TO NOTHING. `buildResourceToNameMap` only ever contains
* ROOT plus the *part-0* resource of each listed node
* (`lib/roles/resolverRoles.ts:138-141,147-161`):
*
* ```
  computeNameResource(name) = keccak256(namehash(name) ‖ bytes32(0))
  ```
*
* so any role held at a resource with part != 0 — which is how the resolver
* scopes a role to a single text key or a single coin type — is absent from the
* map, and so is any node the `nodes` list does not currently include (the name
* was re-pointed at another resolver, indexer lag, or pagination).
* `groupRolesByAccount` then drops it: `resourceToName?.get(resource) ?? null`,
* and only non-null names are added (`resolverRoles.ts:196-206`).
*
* This test drives the REAL `buildResourceToNameMap` / `groupRolesByAccount`
* and the REAL ensjs `grantResolverRolesWriteParameters`. Nothing is stubbed
* except the wallet client's `chain`/`account`, which only decorate the params. \*/ import type { ResolverRole } from '@ensdomains/ensjs/public/v2' import { grantResolverRolesWriteParameters } from '@ensdomains/ensjs/wallet/v2' import { describe, expect, it } from 'vitest' import { encodePacked, type Hex, keccak256, namehash, toHex, } from 'viem' import { packetToBytes } from 'viem/ens' import { buildResourceToNameMap, computeNameResource, groupRolesByAccount, } from '@/lib/roles/resolverRoles'

// happy-dom swallows console.log under vitest; write to stdout directly. const log = (m: string) => process.stdout.write(m + String.fromCharCode(10))

const RESOLVER = '0x1c7d4b196cb0c7b01d743fbc6116a902379c7238' as const const ATTACKER = '0x5fa3a46a6b7f4e374e9671b9dd946d80958c9e47' as const

/\*\* The victim's name, which IS listed under the shared resolver. */ const VICTIM\_NAME = 'victim.eth' /*\* A second name on the same resolver — collateral of a root grant. */ const OTHER\_NAME = 'treasury.eth' /*\* The attacker's name, also on this resolver. \*/ const ATTACKER\_NAME = 'attacker.eth'

/\*\*

* `PermissionedResolverLib.resource(node, part)` for part != 0 — the shape the
* resolver uses for a role scoped to ONE text key. `computeNameResource` is the
* part-0 case of exactly this. \*/ const resourceWithPart = (name: string, part: Hex): string => keccak256(encodePacked(\['bytes32', 'bytes32'], \[namehash(name), part]))

/\*\* keccak of a text key, used as the `part` for a key-scoped role. \*/ const textKeyPart = (key: string) => keccak256(toHex(key))

/\*\* A minimal wallet client — ensjs only reads `chain` and `account`. \*/ const walletClient = { chain: { id: 11155111 }, account: { address: ATTACKER }, } as unknown as Parameters\[0]

/\*\* `ResolverRolesSidebar.tsx:169`, reproduced verbatim. \*/ const roleNameFromSidebar = (resolvedNames: readonly string\[]) => resolvedNames.find((n) => n !== '(root)') ?? ''

/\*\* `grantResolverRoles.ts:45-60`, reproduced verbatim. \*/ const buildGrantParams = (name: string, roles: ResolverRole\[]) => grantResolverRolesWriteParameters( walletClient, name === '' ? { resolverAddress: RESOLVER, targetAccount: ATTACKER, scope: 'root' as const, roles, } : { resolverAddress: RESOLVER, targetAccount: ATTACKER, scope: 'name' as const, name, roles, }, )

/\*\* The resolver's node list as the indexer supplies it. \*/ const NODES = \[ { name: VICTIM\_NAME }, { name: OTHER\_NAME }, { name: ATTACKER\_NAME }, ]

describe('resolver role scope collapses to ROOT when it cannot be resolved', () => { it('control: a part-0 row resolves, and saving it grants NAME scope', () => { const map = buildResourceToNameMap(NODES)

```
const [group] = groupRolesByAccount(
  [
    {
      account: ATTACKER,
      resource: computeNameResource(VICTIM_NAME).toLowerCase(),
      roleBitmap: '0x1',
    },
  ],
  map,
)

// The UI knows the scope, so it renders it and the calldata is name-scoped.
expect(group.resolvedNames).toEqual([VICTIM_NAME])

const name = roleNameFromSidebar(group.resolvedNames)
expect(name).toBe(VICTIM_NAME)

const params = buildGrantParams(name, ['ROLE_SET_ADDR'])
expect(params.functionName).toBe('authorizeNameRoles')
// ...and it targets exactly that one name.
expect(params.args[0]).toBe(toHex(packetToBytes(VICTIM_NAME)))

log(`  [control]   resolvedNames=${JSON.stringify(group.resolvedNames)} -> authorizeNameRoles(${VICTIM_NAME})`)
```

})

it('a key-scoped row (part != 0) resolves to NOTHING and grants ROOT', () => { const map = buildResourceToNameMap(NODES)

```
// A role scoped to the single text key "com.twitter" on the attacker's own
// name. Its resource has part = keccak("com.twitter"), so it is NOT the
// part-0 value the map was built from.
const keyScoped = resourceWithPart(
  ATTACKER_NAME,
  textKeyPart('com.twitter'),
).toLowerCase()
expect(map.has(keyScoped)).toBe(false)

const [group] = groupRolesByAccount(
  [{ account: ATTACKER, resource: keyScoped, roleBitmap: '0x1' }],
  map,
)

// The UI cannot name the scope...
expect(group.resolvedNames).toEqual([])

// ...so ResolverRolesSidebar.tsx:391 renders NO scope line at all.
const scopeLineRendered = group.resolvedNames.length > 0
expect(scopeLineRendered).toBe(false)

// ...and line 169 turns "unknown" into the ROOT sentinel.
const name = roleNameFromSidebar(group.resolvedNames)
expect(name).toBe('')

// The admin clicks Save on one account's row. This is what gets signed:
const params = buildGrantParams(name, ['ROLE_SET_ADDR'])
expect(params.functionName).toBe('grantRootRoles')

// No name anywhere in the calldata — the grant is resolver-wide.
expect(params.args).toHaveLength(2)
expect(params.args[1]).toBe(ATTACKER)

log('  [ESCALATED] resolvedNames=[] -> grantRootRoles(bitmap, attacker) — ALL names')
```

})

it('an unlisted node (re-pointed / indexer lag) does the same', () => { // The attacker re-points attacker.eth at a different resolver, so the // indexer no longer lists it here. Their existing role row remains. const mapWithoutAttacker = buildResourceToNameMap(\[ { name: VICTIM\_NAME }, { name: OTHER\_NAME }, ])

```
const [group] = groupRolesByAccount(
  [
    {
      account: ATTACKER,
      resource: computeNameResource(ATTACKER_NAME).toLowerCase(),
      roleBitmap: '0x1',
    },
  ],
  mapWithoutAttacker,
)

expect(group.resolvedNames).toEqual([])
expect(roleNameFromSidebar(group.resolvedNames)).toBe('')
expect(buildGrantParams('', ['ROLE_SET_ADDR']).functionName).toBe('grantRootRoles')

log('  [ESCALATED] unlisted node -> grantRootRoles — no attacker action needed beyond re-pointing')
```

})

it('the root grant is not scoped to the row: it covers every listed name', () => { // Demonstrate the blast radius concretely — the name-scoped call encodes a // specific name, the root call encodes none, so it cannot exclude any of // the resolver's names. const scoped = buildGrantParams(VICTIM\_NAME, \['ROLE\_SET\_ADDR']) const rootWide = buildGrantParams('', \['ROLE\_SET\_ADDR'])

```
const scopedTargets = [VICTIM_NAME, OTHER_NAME, ATTACKER_NAME].filter(
  (n) => scoped.args[0] === toHex(packetToBytes(n)),
)
expect(scopedTargets).toEqual([VICTIM_NAME])

// The root variant carries no name at all.
expect(
  [VICTIM_NAME, OTHER_NAME, ATTACKER_NAME].some((n) =>
    (rootWide.args as readonly unknown[]).includes(
      toHex(packetToBytes(n)),
    ),
  ),
).toBe(false)

log('  [blast]     authorizeNameRoles targets 1 name; grantRootRoles targets all 3')
```

}) })
