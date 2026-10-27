# 91062 w a critical scope collapse in the resolver roles editor causes role writes to target the resolver root resource granting authority over every name it serves and making remove user repor&#x20;

**Submitted on Aug 29th 2026 at 09:49:16 UTC by @Nyko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91062
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer's resolver-roles editor recovers the scope of a role write with `resolvedNames.find((n) => n !== '(root)') ?? ''`. That empty-string fallback is not "no scope": two files later it selects `scope: 'root'`, the resolver's **root resource** — authority over **every name that resolver serves**. The operator is shown nothing about it: the Name column renders `null`, the scope line hides itself in exactly this case, and the confirmation modal reads only "Save resolver role changes". The same collapse makes **"Remove user"** confirm on-chain and report success while leaving the account's root roles intact. In production this turns a grant intended for one name into resolver-wide authority for the grantee, and turns a revocation an operator believes is complete into a partial one they will not retry.

## Vulnerability Details

### 1 — Roles are grouped per account, and each role's name is recovered by a raw string lookup

`apps/portal/src/lib/roles/resolverRoles.ts` — `groupRolesByAccount`:

```ts
const resolvedName = resourceToName?.get(resource) ?? null
…
existing.decodedRoles.push(...decoded)                        // role bits are unioned regardless
if (resolvedName) existing.resolvedNames.add(resolvedName)    // the name is added ONLY on a hit
```

`buildResourceToNameMap` in the same file only ever indexes `computeNameResource(name)` — that is `PermissionedResolverLib.resource(node, 0)`, **part 0** — and skips nodes whose `name` is null (`if (node.name)`). So any role whose resource is not the part-0 resource of a node in the fetched page contributes **nothing** to `resolvedNames`, while its role bits are still merged into the row and rendered as editable.

The codebase already documents this exact hazard, and fixes it one file away in the same directory. `apps/portal/src/lib/roles/filterEventsByResource.ts`:

```ts
// Resources are uint256 values, so compare numerically — the same value can be
// encoded differently (e.g. ROOT_RESOURCE as "0x0" from the indexer vs the
// 32-byte-padded zero callers build). Falls back to case-insensitive string
// compare for any non-numeric resource.
const resourcesMatch = (a: string, b: string): boolean => {
  try { return BigInt(a) === BigInt(b) } catch { return a.toLowerCase() === b.toLowerCase() }
}
```

`groupRolesByAccount` does precisely what that comment warns against. And the correct answer is already on the wire and discarded: `useResolverOverview.ts` selects `name` inside the `roles` selection, and the schema types it (`EacRoleAssignment.name`) — the grouping ignores that field and re-derives the name by hashing instead.

### 2 — The sidebar collapses the scope to one arbitrary name

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169`

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`roleName` is the `name:` passed to **both** writes — `:299` (save changes) and `:318` (remove user). `find()` returns the first non-root entry, which is arbitrary when several are present, and falls through to `''` when there is none.

### 3 — An empty name selects ROOT, not "no scope"

`apps/portal/src/features/resolver/helpers/grantResolverRoles.ts` (identical branch in `revokeResolverRoles.ts`):

```ts
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

This is not a cosmetic flag. Resolved against `ensjs-abi@e96662c`, the commit pinned in `pnpm-workspace.yaml:10-11`:

| scope    | function                                         | selector     | resource                |
| -------- | ------------------------------------------------ | ------------ | ----------------------- |
| `'root'` | `grantRootRoles(uint256,address)`                | `0x072d5d77` | `0`                     |
| `'name'` | `authorizeNameRoles(bytes,uint256,address,bool)` | `0xbbd9abb5` | `keccak256(namehash‖0)` |

Different function, different resource.

### 4 — There is no guard

`handleSaveChanges` gates only on `!selectedAccount || isPending`; `handleRemoveUser` adds only `decodedRoles.length === 0`. Nothing in the component reads `resolvedNames` except the scope line that hides itself when the array is empty. `ResolverRolesSidebar.tsx` has **no test file anywhere in the repository**.

### 5 — The operator is never warned, at any of the three display layers

| Layer               | Code                                                                   | Behaviour when `resolvedNames` is empty |
| ------------------- | ---------------------------------------------------------------------- | --------------------------------------- |
| Table "Name" column | `ResolverRolesTable.tsx:69-70` — `if (names.length === 0) return null` | renders nothing                         |
| Sidebar scope line  | `ResolverRolesSidebar.tsx:391` — gated on `resolvedNames.length > 0`   | hidden                                  |
| Confirmation modal  | `:343-348` — `'Save resolver role changes'` / `'Update roles for 0x…'` | no scope shown                          |

The one string containing `(root)` reaches only the transaction `description`, which `TransactionModal` never renders. Note the trap: when `(root)` **is** present the sidebar honestly prints "Global roles (all names)" (`:393`) — the warning exists and is suppressed in exactly the case that escalates.

**Sibling-flow asymmetry, one screen away:** `ResolverAddUserSheet.tsx` defines `ROOT_NODE_VALUE = ''` with an explanatory comment (`:39`), offers a visible "Root (all nodes)" option (`:253`), and prints `(root)` in its **confirmation modal** (`:321`). The roles sidebar prints `Update roles for 0x…`.

**Reachability of the empty case**, strongest first:

1. **part-scoped resources** (`text` / `addr` grants) that `computeNameResource(part = 0)` can never produce — the indexer models these as `NamedResourceEntry` (`packages/indexer/graphql.gen.ts:381-390`), a field the Explorer queries nowhere;
2. a raw `namehash` with no `normalize`, while `useNameHistoryTimeline.ts:225-234` normalizes first;
3. a nullable `node.name`, skipped by the code's own `if (node.name)` guard;
4. pagination — `nodes` is requested with no `first:` argument and the server defaults to 100, which `useV2NameHistory.ts:53-56` states verbatim.

## Impact Details

**Variant A — a revocation that reports success while root authority survives.** This variant needs no special conditions: both preconditions are creatable from the Explorer's own Add-user sheet. Give an account a **Root (all nodes)** grant and a name-scoped grant, so `resolvedNames` is `['(root)', 'foo.eth']`. The row displays **"Global roles (all names)"**. The Remove dialog states the user will be *"removed from all roles… cannot be undone"*. But `find()` returns `'foo.eth'`, so the write is `authorizeNameRoles(foo.eth, …, false)` — name-scoped only. **The root roles survive and the interface reports success.** An operator revoking a compromised key or a departing employee believes the access is gone. It is not, and the false confirmation is what stops them retrying.

**Variant B — silent escalation to the resolver root.** With an empty `resolvedNames`, a grant intended for one name is written to the root resource. A contractor given `ROLE_SET_TEXT` on a single name ends up able to repoint the `addr` record of **every name the resolver serves** — for example `treasury.eth`. No screen in the flow states the scope, so the operator has no opportunity to notice before signing.

**On the "privileged addresses" exclusion:** it does not apply here. The editor is reachable only to root-authority holders (`routes/resolver/$address/roles/index.tsx:41-50` gates on `hasRootRoles`), which means the privileged party is the **victim** of the mis-scoped write. The beneficiary is an unprivileged grantee who receives authority nobody intended to give them.

**Severity.** Filed at the tier of the selected in-scope impact — *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user … submits a transaction to an unintended recipient."* The mechanism is exactly that. Stating the counter-argument openly rather than leaving it for triage: "recipient" reads most naturally as a counterparty receiving assets, and here **no asset moves** — what moves is *authority*. A reviewer reading that line strictly could place this at High, and I would not dispute a considered downgrade. What is not in question is the mechanism: the displayed scope and the signed scope differ, and the difference is authority over every name the resolver serves.

**Why this is not covered by the published known issues.** Nothing in that document touches `groupRolesByAccount`, resolver-role scope, or `ResolverRolesSidebar`. QA-01, WEB-1257 and WEB-1305 describe flows that **error** or leave state detached after a transfer — here nothing errors; the transaction succeeds and writes to the wrong resource. WEB-1249 concerns subname creation replacing a child registry (different feature, different mechanism). WEB-716 is a missing UI affordance for replacing a subregistry. WEB-1255 is the closest structurally — a wrongly-defaulted scalar — but there the value is **presentational** (which grace-period text is shown), whereas here the default **selects the on-chain resource the role is written to**. The known-issues document states that a demonstrated security consequence beyond "merely erroring" is a new finding.

**Recommended fix.** Use the `name` the indexer already returns on each role, or compare resources numerically exactly as `filterEventsByResource.ts` already does. Key each write to the resource its role actually came from rather than to a name recovered by `find()`; make the empty case fail closed so `''` can never mean root; and render the resolved scope in the confirmation modal, as the Add-user sheet already does one screen away.

## References

* `apps/portal/src/lib/roles/resolverRoles.ts` — `groupRolesByAccount`, `buildResourceToNameMap`, `computeNameResource`
* `apps/portal/src/lib/roles/filterEventsByResource.ts` — the same hazard, documented and correctly handled
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx` — `:169`, `:299`, `:318`, `:343-348`, `:391-393`
* `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx` — `:69-70`, `:103`
* `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts`, `revokeResolverRoles.ts`
* `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx` — `:39`, `:253`, `:321`
* `apps/portal/src/features/resolver/hooks/useResolverOverview.ts` — the `roles { … name }` selection
* `apps/portal/src/routes/resolver/$address/roles/index.tsx:41-50` — the `hasRootRoles` gate
* `packages/indexer/graphql.gen.ts:381-390` — `NamedResourceEntry`
* `pnpm-workspace.yaml:10-11` — the pinned `ensjs` / `ensjs-abi` commit used to resolve the selectors

## Proof of Concept

### A. Runnable reproduction — no install, no chain, no network

Save as `poc.mjs`, run `node poc.mjs` (Node 22). It exits 0 on reproduction. The functions are copied verbatim from the audited revision and annotated with their source paths; only `decodeResolverRoleBitmap` is stubbed, because the defect concerns `resolvedNames` and never the bitmap.

```js
// VERBATIM · apps/portal/src/lib/roles/resolverRoles.ts
const ROOT_RESOURCE = '0x0'

const buildResourceToNameMap = (nodes, computeNameResource) => {
  const map = new Map()
  map.set(ROOT_RESOURCE, '(root)')
  for (const node of nodes) {
    if (node.name) {
      map.set(computeNameResource(node.name).toLowerCase(), node.name)
    }
  }
  return map
}

const groupRolesByAccount = (roles, resourceToName) => {
  const grouped = new Map()
  for (const role of roles) {
    const account = role.account.toLowerCase()
    const resource = role.resource.toLowerCase()
    const decoded = decodeResolverRoleBitmap(role.roleBitmap)
    const resolvedName = resourceToName?.get(resource) ?? null   // raw string lookup
    const existing = grouped.get(account)
    if (existing) {
      existing.roles.push(role)
      existing.decodedRoles.push(...decoded)                     // roles UNIONED regardless
      if (resolvedName) existing.resolvedNames.add(resolvedName) // name added ONLY on a hit
    } else {
      const names = new Set()
      if (resolvedName) names.add(resolvedName)
      grouped.set(account, { account, resolvedNames: names, roles: [role], decodedRoles: [...decoded] })
    }
  }
  return Array.from(grouped.values()).map((g) => ({ ...g, resolvedNames: Array.from(g.resolvedNames) }))
}

// stubbed: the defect is in `resolvedNames`, never in the bitmap
const decodeResolverRoleBitmap = () => ['ROLE_SET_TEXT']

// VERBATIM · ResolverRolesSidebar.tsx:169
const sidebarRoleName = (resolvedNames) => resolvedNames.find((n) => n !== '(root)') ?? ''

// VERBATIM · grantResolverRoles.ts (revokeResolverRoles.ts is identical)
const scopeForName = (name) => (name === '' ? 'root' : 'name')

const CALL = {
  root: 'grantRootRoles(uint256,address)               selector 0x072d5d77   resource 0',
  name: 'authorizeNameRoles(bytes,uint256,address,bool) selector 0xbbd9abb5   resource keccak256(namehash‖0)',
}

// SCENARIO: a part-scoped (text) grant — a resource computeNameResource(part=0) can never produce.
const fakeHash = (s) => '0x' + Buffer.from(s).toString('hex').padEnd(64, '0').slice(0, 64)
const computeNameResource = (name) => fakeHash('part0:' + name)

const nodes = [{ name: 'foo.eth' }, { name: 'bar.eth' }]
const roles = [{
  account: '0xAttacker000000000000000000000000000000001',
  resource: fakeHash('part2:foo.eth'),
  roleBitmap: '0x1',
}]

const map = buildResourceToNameMap(nodes, computeNameResource)
const [row] = groupRolesByAccount(roles, map)
const roleName = sidebarRoleName(row.resolvedNames)
const scope = scopeForName(roleName)

const ok = (c) => (c ? 'PASS' : 'FAIL')
console.log('\nFINDING — resolver-roles scope collapses to ROOT\n' + '-'.repeat(72))
console.log('operator intent          : grant ROLE_SET_TEXT on ONE name')
console.log('row.resolvedNames        :', JSON.stringify(row.resolvedNames))
console.log('row.decodedRoles         :', JSON.stringify(row.decodedRoles))
console.log('sidebar shows scope line : ', row.resolvedNames.length > 0 ? 'yes' : 'NO  (gated on resolvedNames.length > 0, ResolverRolesSidebar.tsx:391)')
console.log('table Name column        : ', row.resolvedNames.length > 0 ? row.resolvedNames.join(', ') : 'NOTHING  (returns null, ResolverRolesTable.tsx:69-70)')
console.log('confirmation modal       :  "Save resolver role changes" / "Update roles for 0x..."  (no scope)')
console.log('-'.repeat(72))
console.log('roleName passed to write :', JSON.stringify(roleName))
console.log('resulting scope          :', scope.toUpperCase())
console.log('calldata                 :', CALL[scope])
console.log('-'.repeat(72))
console.log(ok(row.resolvedNames.length === 0), 'the scope is invisible to the operator')
console.log(ok(roleName === ''),                'the name collapses to the empty string')
console.log(ok(scope === 'root'),               'the write targets the resolver ROOT resource')
process.exit(row.resolvedNames.length === 0 && roleName === '' && scope === 'root' ? 0 : 1)
```

Observed output:

```
row.resolvedNames        : []
row.decodedRoles         : ["ROLE_SET_TEXT"]
sidebar shows scope line :  NO  (gated on resolvedNames.length > 0, ResolverRolesSidebar.tsx:391)
table Name column        :  NOTHING  (returns null, ResolverRolesTable.tsx:69-70)
confirmation modal       :  "Save resolver role changes" / "Update roles for 0x..."  (no scope)
roleName passed to write : ""
resulting scope          : ROOT
calldata                 : grantRootRoles(uint256,address)               selector 0x072d5d77   resource 0
```

### B. Verification against the shipped source (recommended)

This test **imports** the real functions — nothing is reimplemented — and includes a control case proving the map resolves a part-0 resource correctly, so the failure cannot be an artefact of the test. Save as `apps/portal/src/lib/roles/finding1.test.ts` and run `pnpm test:portal`.

```ts
import { describe, expect, it } from 'vitest'
import { buildResourceToNameMap, computeNameResource, groupRolesByAccount } from './resolverRoles'

// ResolverRolesSidebar.tsx:169 — verbatim
const sidebarRoleName = (resolvedNames: readonly string[]) =>
  resolvedNames.find((n) => n !== '(root)') ?? ''
// grantResolverRoles.ts / revokeResolverRoles.ts — verbatim branch
const scopeForName = (name: string) => (name === '' ? 'root' : 'name')

describe('resolver-roles scope collapses to ROOT', () => {
  it('writes to the resolver ROOT when a role resource is not part-0 of a fetched node', () => {
    const nodes = [{ name: 'foo.eth' }, { name: 'bar.eth' }]
    const map = buildResourceToNameMap(nodes)

    // buildResourceToNameMap only indexes computeNameResource(name), i.e.
    // PermissionedResolverLib.resource(node, part = 0). A text- or addr-scoped grant
    // (part != 0) is a real resource the indexer returns as NamedResourceEntry —
    // a field the Explorer queries nowhere.
    const partScoped = ('0x' + 'ab'.repeat(32)) as `0x${string}`
    expect(map.has(partScoped.toLowerCase())).toBe(false)

    const [row] = groupRolesByAccount(
      [{ account: '0xattacker', resource: partScoped, roleBitmap: '0x1' }],
      map,
    )

    // 1. the operator is shown nothing about the scope
    expect(row.resolvedNames).toEqual([])
    expect(row.decodedRoles.length).toBeGreaterThan(0)   // yet the role IS editable

    // 2. the sidebar collapses the scope to the empty string
    const roleName = sidebarRoleName(row.resolvedNames)
    expect(roleName).toBe('')

    // 3. the empty string selects the ROOT resource, not "no scope"
    expect(scopeForName(roleName)).toBe('root')
  })

  it('control: a part-0 resource resolves correctly, proving the map itself works', () => {
    const nodes = [{ name: 'foo.eth' }]
    const map = buildResourceToNameMap(nodes)
    const [row] = groupRolesByAccount(
      [{ account: '0xok', resource: computeNameResource('foo.eth'), roleBitmap: '0x1' }],
      map,
    )
    expect(row.resolvedNames).toEqual(['foo.eth'])
    expect(scopeForName(sidebarRoleName(row.resolvedNames))).toBe('name')
  })
})
```

### C. Step-by-step reproduction in the application

{% stepper %}
{% step %}

#### As a resolver operator, open the resolver Roles page

As a resolver operator (holding root roles — the only way to reach this screen), open the Explorer → your resolver → **Roles**.
{% endstep %}

{% step %}

#### Create root and name-scoped grants

Using the **Add user** sheet, grant account `0xAAA…` a **Root (all nodes)** role, then grant it a name-scoped role on `foo.eth`.
{% endstep %}

{% step %}

#### Verify the displayed global scope

The Roles table now shows **one** row for `0xAAA…`. Open it. The sidebar reads **"Global roles (all names)"**.
{% endstep %}

{% step %}

#### Remove the user

Press **Remove user**. The dialog states the user will be *"removed from all roles… cannot be undone"*. The confirmation modal shows only `Remove user 0xAAA…` — no scope.
{% endstep %}

{% step %}

#### Sign the transaction

Sign. The write is `authorizeNameRoles(foo.eth, …, false)`. The UI reports the removal succeeded.
{% endstep %}

{% step %}

#### Verify root authority remains

**Re-open the resolver's roles.** `0xAAA…` still holds its root roles — authority over every name the resolver serves — despite the interface having confirmed a complete removal.
{% endstep %}
{% endstepper %}

### Variant B — silent escalation to root

{% stepper %}
{% step %}

#### Open the resolver Roles page

Explorer → your resolver → **Roles**.
{% endstep %}

{% step %}

#### Ensure the account has an unresolved role resource

Ensure `0xAAA…` holds a role whose resource is not the part-0 resource of a node in the fetched page — e.g. a `text`-scoped grant (reachability path 1 above), or a role on a name beyond the server's default page of 100 nodes.
{% endstep %}

{% step %}

#### Observe the missing scope

The row appears with its roles editable, but the **Name column is empty** and the sidebar shows **no scope line**.
{% endstep %}

{% step %}

#### Save a role change

Tick any additional role and press **Save changes**. The modal reads `Save resolver role changes` / `Update roles for 0x…` — no scope.
{% endstep %}

{% step %}

#### Sign the transaction

Sign. `roleName` is `''`, so `grantResolverRoles` takes the `scope: 'root'` branch and the calldata is `grantRootRoles(uint256,address)` (`0x072d5d77`) against resource `0`.
{% endstep %}

{% step %}

#### Verify resolver-wide authority

`0xAAA…` now holds that role over **every name the resolver serves**, and nothing in the flow said so.
{% endstep %}
{% endstepper %}
