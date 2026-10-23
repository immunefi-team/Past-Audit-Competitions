# 93099 w a critical resolver roles the sidebar displays one permission scope and signs a different one

**Submitted on Sep 14th 2026 at 08:51:44 UTC by @spryon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93099
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The resolver roles screen computes the permission scope it **shows** and the permission scope it **signs** by two different rules, from the same row. The scope line joins *every* name the delegate holds roles on; the transaction takes `resolvedNames.find(n => n !== '(root)') ?? ''` — the **first** one, or the empty string. That empty string is simultaneously the sentinel the transaction builders use for "root resource", meaning *every name on this resolver*. So an operator managing delegated permissions can read one scope on screen and sign a different one: a different name, or — when no name resolves — a resolver-wide grant. A second, independent defect in the same file omits a real on-chain role bit from the portal's role table, so that role is invisible in the UI and cannot be cleared by "Remove user".

## Vulnerability Details

### The two rules

Display, [`ResolverRolesSidebar.tsx:391-395`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L395) — joins **all** names:

```tsx
{selectedAccount && resolvedNames.length > 0 && (
  <p className="text-sm text-muted-foreground">
    {resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
  </p>
```

Transaction, [`ResolverRolesSidebar.tsx:169`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L169) — takes **one**, or `''`:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`roleName` is what both mutations are given — [`:297-301`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L297-L301) (save) and [`:316-322`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L316-L322) (remove).

### The overloaded sentinel

`''` is not a neutral fallback. It is the codebase's marker for *root scope*, in both builders — [`grantResolverRoles.ts:47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L47) and [`revokeResolverRoles.ts:47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L47):

```ts
name === '' ? { resolverAddress, targetAccount: account, scope: 'root', roles } : …
```

which ensjs turns into `grantRootRoles` / `revokeRootRoles` rather than `authorizeNameRoles`. The portal's own "Add user" sheet uses the same value deliberately — [`ResolverAddUserSheet.tsx:39`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L39) `const ROOT_NODE_VALUE = ''`, and it is the **default** selection at [`:127`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L127). So `''` genuinely means "every name on this resolver" here, and line 169 can produce it by accident.

### Why a row can carry several names, or none

[`groupRolesByAccount`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L196-L206) keys rows by **account alone**. Roles held on different resources are merged: `decodedRoles` becomes the union, and `resolvedNames` accumulates into a `Set` — but only when the resource is mappable (`if (resolvedName) existing.resolvedNames.add(resolvedName)`).

[`buildResourceToNameMap`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L147-L159) can only map `ROOT_RESOURCE` and the names in the indexer's `nodes` list. Anything else — a `part != 0` resource (ensjs supports `scope: 'text' | 'addr'`), or a node absent from that page — maps to nothing, so it contributes no entry to `resolvedNames`.

### Three divergences, demonstrated

**1. Two names, one row — no preconditions whatsoever.** An account with roles on `a.eth` and `b.eth` on the same resolver renders as a single row. Real PoC output:

```
resolvedNames      : [ 'a.eth', 'b.eth' ]
UI scope line      : Roles scoped to a.eth, b.eth
roleName used      : "a.eth"
decoded args       : [ '0x01610365746800', '1', '0xDDdD…DDDd', 'true' ]   <- a.eth
```

The operator reads two names and signs one. Whatever they intended for `b.eth` does not happen — a revocation silently leaves `b.eth` intact, a grant silently lands on `a.eth`. This requires no exotic setup and is reachable entirely through first-party UI.

**2. UI says global, calldata is scoped.** Root plus name-scoped roles on one account:

```
resolvedNames      : [ '(root)', 'alice.eth' ]
UI scope line      : Global roles (all names)
decoded function   : authorizeNameRoles
decoded args       : [ '0x05616c6963650365746800', '17', '0xDDdD…DDDd', 'false' ]
```

The screen says the delegate holds global roles; "Remove user" emits a name-scoped revoke. The resolver-wide grant survives an operation the operator was told succeeded.

**3. No mappable name → resolver-wide grant, with no scope line shown at all.** When `resolvedNames` is empty, `roleName` collapses to `''` and the scope paragraph is suppressed by the `resolvedNames.length > 0` guard, so the operator is shown *nothing*:

```
resolvedNames      : []
scope line rendered: false
roleName (line 169): ""
decoded function   : grantRootRoles
decoded args       : [ '1', '0xDDdD…DDDd' ]
raw calldata       : 0x072d5d77…
```

A grant the operator believes is scoped to one name is signed as **resolver-wide**. The PoC reproduces this two ways: via a `text`-scoped resource, and via a node simply missing from the indexer's `nodes` page (which needs no exotic grant at all).

### Second, independent defect: `ROLE_SET_DATA` is invisible and unrevokable

The portal hand-copies the resolver role table at [`resolverRoles.ts:24-45`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L24-L45) and **omits `ROLE_SET_DATA` (bit 36)**, which ensjs — the ENS team's own client for this contract — defines. A grep for `36n` or `SET_DATA` in that file returns nothing. Consequences, measured:

```
ensjs RESOLVER_ROLE_SET_DATA    : 68719476736n
portal table has that bit?      : false
on-chain bitmap                 : 68719476737n        (SET_ADDR + SET_DATA)
portal decode                   : [ 'ROLE_SET_ADDR' ] (SET_DATA dropped)
bitmap "Remove user" revokes    : 1n
residual AFTER a "successful"   : 68719476736n
residual === ROLE_SET_DATA?     : true
```

`setData` is raw resolver-storage write for the resource. A delegate who has been "removed" retains the ability to rewrite what the name resolves to — its addr, text and contenthash records — and the UI never showed the role in the first place.

## Impact Details

### Selected impact: transaction argument modification

This is the mapping the PoC demonstrates directly and it is the same one that applies to our confirmed report #92978: the application presents one authorisation scope and signs another. The `name` argument of `authorizeNameRoles` / the choice between that and `grantRootRoles` is the modified argument, and it is produced by the app itself from a rule that does not match what the operator was shown.

### Consequences

**Privilege escalation (divergence 3).** A grant the operator scopes to one name becomes resolver-wide. The delegate obtains the granted roles — e.g. `ROLE_SET_ADDR` — over **every name that resolver serves**, and can repoint the ETH address record of any of them. That reaches "displaying incorrect addresses … leading to misdirected transfers" downstream, since a repointed `addr` record sends funds to an address of the delegate's choosing.

**Failed revocation (divergences 1 and 2).** An operator removing a delegate's access is told it succeeded while the access survives — on `b.eth` in case 1, resolver-wide in case 2. The security-relevant outcome is that a party the operator believes they have de-authorised retains write authority over name records.

**Permanently retained role (the `ROLE_SET_DATA` defect).** Independent of the scope question, `setData` cannot be revoked through this UI at all, because the bitmap the app signs never contains bit 36.

### What is NOT established

* **The on-chain behaviour of `authorizeNameRoles(..., false)` when the account does not hold those bits at that resource.** The contracts are out of scope for this competition and are not in this repository. If that call reverts, divergence 2 degrades from a silent failure to a loud one. **Divergences 1 and 3 do not depend on this** — they are wrong-target and wider-target respectively, not no-op reverts.
* Divergence 3 requires either a `part != 0` resource (ensjs supports `scope: 'text' | 'addr'`; the portal's own UI does not create them) **or** a node missing from the indexer's `nodes` page. The PoC demonstrates both routes; the second needs no unusual grant.
* The PoC executes the real grouping, decoding and calldata-building functions against in-memory inputs. It does not submit on-chain transactions or render React.
* No attacker is required for divergences 1 and 2 — they are an operator being misled by their own tool. Divergence 3 likewise arises from ordinary state.

### Relationship to my other submissions

Disclosed rather than left to be found: I have two confirmed reports in this competition (#92978 portal transfer, #92945 manager renew). Both are **canonicalisation** defects — a raw versus ENSIP-15-normalised name. **This is not that class.** Nothing here normalises, lowercases, hashes or splits a name; every name involved is already canonical. The divergence is between two **EAC resource scopes**, caused by an overloaded `''` sentinel plus account-keyed aggregation. The fix is disjoint: normalising route parameters would not change a single line of behaviour described above.

### Recommended fix

1. **Stop deriving the transaction scope from the display list.** Carry the resource (or an explicit `{scope, name}` pair) on the row and pass *that* to the mutation, rather than re-deriving a name with `.find()`.
2. **Do not overload `''`.** Use a distinct, explicit marker for root scope so an "unresolvable name" can never silently become "every name". Failing to resolve a name should be an error state, not a fallback.
3. **Key rows by `(account, resource)`**, or render one row per resource, so a delegate's permissions on different names are never merged into one editable row.
4. **Derive the role table from ensjs** instead of hand-copying it, which closes the `ROLE_SET_DATA` gap and any future drift.

## References

All links are permalinks pinned to commit [`1c9b47f`](https://github.com/immunefi-team/audit-comp-ens/commit/1c9b47f18fcddd2e864dfe385c4171061c9811ae).

**The divergence**

* [`ResolverRolesSidebar.tsx#L169`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L169) — `.find(…) ?? ''`, the transaction scope
* [`ResolverRolesSidebar.tsx#L391-L395`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L395) — the displayed scope, joining all names
* [`ResolverRolesSidebar.tsx#L297-L301`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L297-L301) — save passes `name: roleName`
* [`ResolverRolesSidebar.tsx#L316-L322`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L316-L322) — remove passes `name: roleName`, `roles: decodedRoles`

**The overloaded sentinel**

* [`grantResolverRoles.ts#L47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L47) — `name === '' ? scope:'root'`
* [`revokeResolverRoles.ts#L47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L47) — same
* [`ResolverAddUserSheet.tsx#L39`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L39) — `ROOT_NODE_VALUE = ''`, default at [`#L127`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L127)

**Why rows carry several names or none**

* [`resolverRoles.ts#L196-L206`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L196-L206) — account-keyed grouping, union of roles, conditional name set
* [`resolverRoles.ts#L147-L159`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L147-L159) — only root and indexed `part == 0` names are mappable

**The missing role bit**

* [`resolverRoles.ts#L24-L45`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L24-L45) — role table, no bit 36

## Proof of Concept

**12/12 assertions pass across two files, offline.** Both import and execute the repository's real code — `groupRolesByAccount`, `buildResourceToNameMap`, `decodeResolverRoleBitmap`, `computeNameResource`, and the real `grantResolverRolesWriteParameters` / `revokeResolverRolesWriteParameters` from ensjs — and assert on **decoded calldata**, including the real selectors `grantRootRoles` `0x072d5d77` and `authorizeNameRoles` `0xbbd9abb5`.

### Run

```bash
cd apps/portal
cp resolver-role-scope-collapse.poc.test.ts src/__v1.test.ts
cp role-set-data-invisible.poc.test.ts      src/__v2.test.ts
./node_modules/.bin/vitest run src/__v1.test.ts src/__v2.test.ts --reporter=verbose
rm src/__v1.test.ts src/__v2.test.ts
```

### What each assertion establishes

| Test                                | Establishes                                                                                              |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------- |
| wrong-name target, two names        | **Divergence 1** — UI "Roles scoped to a.eth, b.eth", calldata targets `a.eth` only                      |
| inverse: UI GLOBAL, calldata scoped | **Divergence 2** — UI "Global roles (all names)", calldata `authorizeNameRoles(alice.eth, …, false)`     |
| HOP 1–4 scope collapse              | **Divergence 3** — unmappable resource → `roleName === ''` → `grantRootRoles`, scope line not rendered   |
| node missing from `nodes` page      | Divergence 3 again, with no exotic scope required                                                        |
| CONTROL (single name)               | A plain row correctly encodes scoped `authorizeNameRoles` — proving a divergence, not a blanket mismatch |
| SET\_DATA HOP 1–4                   | Bit 36 absent from the table, dropped on decode, and unclearable by "Remove user"                        |
| CONTROL (known roles)               | A table-known bitmap round-trips losslessly — residual `0n`                                              |

### Observed output

```
RUN  v4.1.5 /run/media/hbt/work/ensdomains/audit-comp-ens/apps/portal

stdout | src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — scope collapse to ROOT > HOP 1: a text-key-scoped grant produces a resource buildResourceToNameMap can never map
name-scoped resource (part=0): 0x3006408d6a4e95265b62b102a4e44051f3e56ac21ac9ab05d7dde73b0167d9b5
text-scoped resource (avatar): 0xeefe53aa46b2a6975e416322c345f816af0925b7e0d311c3e0541b25674b9fcf
map knows name-scoped        : victim.eth
map knows text-scoped        : undefined

stdout | src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — scope collapse to ROOT > HOP 2: the row for that delegate carries NO resolved name, so the sidebar shows no scope at all
resolvedNames      : []
decodedRoles       : [ 'ROLE_SET_TEXT' ]
scope line rendered: false
roleName (line 169): ""

stdout | src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — scope collapse to ROOT > HOP 3+4: ticking one checkbox emits grantRootRoles — resolver-wide, not victim.eth
rolesToGrant       : [ 'ROLE_SET_ADDR' ]
to                 : 0xresolver00000000000000000000000000000001
raw calldata       : 0x072d5d770000000000000000000000000000000000000000000000000000000000000001000000000000000000000000dddddddddddddddddddddddddddddddddddddddd
decoded function   : grantRootRoles
decoded args       : [ '1', '0xDDdDddDdDdddDDddDDddDDDDdDdDDdDDdDDDDDDd' ]
scoped calldata    : 0xbbd9abb500000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000001000000000000000000000000dddddddddddddddddddddddddddddddddddddddd0000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000c0676696374696d03657468000000000000000000000000000000000000000000

stdout | src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — scope collapse to ROOT > same collapse with NO exotic scope: a node simply missing from the indexer `nodes` page
resolvedNames      : []
roleName           : ""
decoded function   : revokeRootRoles [ '1', '0xDDdDddDdDdddDDddDDddDDDDdDdDDdDDdDDDDDDd' ]

stdout | src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — scope collapse to ROOT > CONTROL: a plain single-name row encodes the scoped authorizeNameRoles call
control function   : authorizeNameRoles
control args       : [
  '0x0676696374696d0365746800',
  '16',
  '0xDDdDddDdDdddDDddDDddDDDDdDdDDdDDdDDDDDDd',
  'true'
]

stdout | src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — wrong-name target when one account holds two names > an edit the operator makes while reading "b.eth" is encoded against a.eth
resolvedNames      : [ 'a.eth', 'b.eth' ]
UI scope line      : Roles scoped to a.eth, b.eth
roleName used      : "a.eth"
decoded args       : [
  '0x01610365746800',
  '1',
  '0xDDdDddDdDdddDDddDDddDDDDdDdDDdDDdDDDDDDd',
  'true'
]

stdout | src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — the inverse: UI says GLOBAL, calldata is scoped > "Remove user" on a row that holds root + per-name roles leaves the GLOBAL grant intact
resolvedNames      : [ '(root)', 'alice.eth' ]
decodedRoles UNION : [ 'ROLE_SET_ADDR', 'ROLE_SET_TEXT' ]
UI scope line      : Global roles (all names)
roleName used      : "alice.eth"
raw calldata       : 0xbbd9abb500000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000011000000000000000000000000dddddddddddddddddddddddddddddddddddddddd0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000b05616c6963650365746800000000000000000000000000000000000000000000
decoded function   : authorizeNameRoles
decoded args       : [
  '0x05616c6963650365746800',
  '17',
  '0xDDdDddDdDdddDDddDDddDDDDdDdDDdDDdDDDDDDd',
  'false'
]

 ✓ src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — scope collapse to ROOT > HOP 1: a text-key-scoped grant produces a resource buildResourceToNameMap can never map 7ms
 ✓ src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — scope collapse to ROOT > HOP 2: the row for that delegate carries NO resolved name, so the sidebar shows no scope at all 3ms
 ✓ src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — scope collapse to ROOT > HOP 3+4: ticking one checkbox emits grantRootRoles — resolver-wide, not victim.eth 7ms
 ✓ src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — scope collapse to ROOT > same collapse with NO exotic scope: a node simply missing from the indexer `nodes` page 2ms
 ✓ src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — scope collapse to ROOT > CONTROL: a plain single-name row encodes the scoped authorizeNameRoles call 2ms
 ✓ src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — wrong-name target when one account holds two names > an edit the operator makes while reading "b.eth" is encoded against a.eth 2ms
 ✓ src/resolver-role-scope-collapse.poc.test.ts > portal resolver roles — the inverse: UI says GLOBAL, calldata is scoped > "Remove user" on a row that holds root + per-name roles leaves the GLOBAL grant intact 4ms

 Test Files  1 passed (1)
      Tests  7 passed (7)
   Start at  09:12:30
   Duration  2.62s (transform 395ms, setup 121ms, import 1.93s, tests 29ms, environment 339ms)
```

```
RUN  v4.1.5 /run/media/hbt/work/ensdomains/audit-comp-ens/apps/portal

stdout | src/__verify-setdata.test.ts > portal resolver roles — ROLE_SET_DATA is invisible and unrevokable > HOP 1: ensjs defines ROLE_SET_DATA at bit 36; the portal table does not
ensjs RESOLVER_ROLE_SET_DATA      : 68719476736n
portal table has that bit?       : false
portal table keys               : [
  'ROLE_SET_ADDR',
  'ROLE_SET_TEXT',
  'ROLE_SET_CONTENTHASH',
  'ROLE_SET_PUBKEY',
  'ROLE_SET_ABI',
  'ROLE_SET_INTERFACE',
  'ROLE_SET_NAME',
  'ROLE_SET_ALIAS',
  'ROLE_CLEAR',
  'ROLE_UPGRADE'
]

stdout | src/__verify-setdata.test.ts > portal resolver roles — ROLE_SET_DATA is invisible and unrevokable > HOP 2: a bitmap carrying SET_ADDR + SET_DATA decodes to SET_ADDR only
on-chain bitmap                 : 68719476737n
portal decodeResolverRoleBitmap : [ 'ROLE_SET_ADDR' ]

stdout | src/__verify-setdata.test.ts > portal resolver roles — ROLE_SET_DATA is invisible and unrevokable > HOP 3: groupRolesByAccount propagates the lossy decode into the table row
row.decodedRoles                : [ 'ROLE_SET_ADDR' ]

stdout | src/__verify-setdata.test.ts > portal resolver roles — ROLE_SET_DATA is invisible and unrevokable > HOP 4: the revoke bitmap "Remove user" signs cannot clear bit 36
bitmap held on-chain            : 68719476737n
bitmap "Remove user" revokes    : 1n
residual AFTER a "successful"   : 68719476736n
residual === ROLE_SET_DATA?     : true

stdout | src/__verify-setdata.test.ts > portal resolver roles — ROLE_SET_DATA is invisible and unrevokable > CONTROL: a bitmap with only table-known roles round-trips losslessly
control held                    : 17n
control revoked                 : 17n
control residual                : 0n

 ✓ src/__verify-setdata.test.ts > portal resolver roles — ROLE_SET_DATA is invisible and unrevokable > HOP 1: ensjs defines ROLE_SET_DATA at bit 36; the portal table does not 6ms
 ✓ src/__verify-setdata.test.ts > portal resolver roles — ROLE_SET_DATA is invisible and unrevokable > HOP 2: a bitmap carrying SET_ADDR + SET_DATA decodes to SET_ADDR only 2ms
 ✓ src/__verify-setdata.test.ts > portal resolver roles — ROLE_SET_DATA is invisible and unrevokable > HOP 3: groupRolesByAccount propagates the lossy decode into the table row 1ms
 ✓ src/__verify-setdata.test.ts > portal resolver roles — ROLE_SET_DATA is invisible and unrevokable > HOP 4: the revoke bitmap "Remove user" signs cannot clear bit 36 1ms
 ✓ src/__verify-setdata.test.ts > portal resolver roles — ROLE_SET_DATA is invisible and unrevokable > CONTROL: a bitmap with only table-known roles round-trips losslessly 1ms

 Test Files  1 passed (1)
      Tests  5 passed (5)
   Start at  09:16:51
   Duration  1.84s (transform 80ms, setup 171ms, import 765ms, tests 14ms, environment 600ms)
```

### Complete PoC source — `resolver-role-scope-collapse.poc.test.ts`

```ts
/**
 * HERMETIC PoC — portal `/resolver/$address/roles` role-scope collapse.
 *
 * Mechanism (NOT canonicalisation — no string normalisation anywhere in the
 * chain; every name here is already NFC/ENSIP-15 canonical):
 *
 *   apps/portal/src/lib/roles/resolverRoles.ts:179-221  groupRolesByAccount()
 *     merges EVERY EAC resource an account holds into ONE table row.
 *     `resolvedNames` is a Set of the names it could reverse-map
 *     (buildResourceToNameMap, :147-160, only knows resources with part = 0),
 *     `decodedRoles` is the UNION of the bitmaps across all those resources.
 *
 *   apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169
 *     const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
 *
 *   apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:46-61
 *   apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-60
 *     name === '' ? scope 'root' : scope 'name'
 *     -> grantRootRoles / revokeRootRoles  (GLOBAL, every name on the resolver)
 *        vs authorizeNameRoles(dnsEncode(name), ...)  (scoped)
 *
 * So the single sentinel '' means BOTH "the root resource" AND "we could not
 * reverse-map any resource for this row". Every unmappable resource — a
 * text-key- or coin-scoped grant (part != 0), or a node the indexer's `nodes`
 * page did not return — silently becomes a RESOLVER-WIDE grant/revoke, and the
 * sidebar's scope line (`ResolverRolesSidebar.tsx:391-397`) is not rendered at
 * all in that case because it is guarded on `resolvedNames.length > 0`.
 *
 * The inverse also holds: a row that mixes a root grant with a per-name grant
 * DISPLAYS "Global roles (all names)" but WRITES at the per-name scope, so
 * "Remove user" leaves the global grant intact.
 *
 * Everything below imports the real repo modules. No network, no chain.
 */
import { computeResolverResource } from '@ensdomains/ensjs/public/v2'
import {
  decodeFunctionData,
  keccak256,
  namehash,
  toBytes,
  type Address,
} from 'viem'
import { describe, expect, it } from 'vitest'

/**
 * ensjs `textPart` (dist/utils/v2/roles/resolverResource.js:26-28) — it is not
 * re-exported from any public entrypoint, so it is restated verbatim here:
 * `PermissionedResolverLib.partHash(string) = keccak256(bytes(key))`.
 */
const textPart = (key: string) => keccak256(toBytes(key))

import { prepareGrantResolverRolesTransaction } from '@/features/resolver/helpers/grantResolverRoles'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'
import type { ResolverNode } from '@/features/resolver/hooks/useResolverOverview'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  type ResolverRoleKey,
} from '@/lib/roles/resolverRoles'
import {
  computeRoleChanges,
  roleToPermissions,
} from '@/lib/roles/rolesToPermissions'

// ---------------------------------------------------------------------------
// Fixtures
// ---------------------------------------------------------------------------

const RESOLVER = '0xREsoLVER00000000000000000000000000000001'.toLowerCase() as Address
const OPERATOR = '0x1111111111111111111111111111111111111111' as Address
const DELEGATE = '0xdddddddddddddddddddddddddddddddddddddddd' as Address
const CHAIN_ID = 11155111

// A minimal viem WalletClient shape — `prepare*Transaction` only reads
// `.account` and `.chain`.
const walletClient = {
  account: { address: OPERATOR, type: 'json-rpc' },
  chain: { id: CHAIN_ID, name: 'sepolia' },
} as never

const ROLE_SET_ADDR_BIT = 1n << 0n
const ROLE_SET_TEXT_BIT = 1n << 4n

const hex32 = (v: bigint) => `0x${v.toString(16).padStart(64, '0')}`
const ROOT_RESOURCE = hex32(0n)

/** Exactly the two lines the sidebar runs (ResolverRolesSidebar.tsx:167-169). */
const sidebarScope = (resolvedNames: readonly string[]) => ({
  roleName: resolvedNames.find((n) => n !== '(root)') ?? '',
  // ResolverRolesSidebar.tsx:391-397 — the whole <p> is behind this guard.
  scopeLineRendered: resolvedNames.length > 0,
  scopeLineText:
    resolvedNames.length === 0
      ? null
      : resolvedNames.includes('(root)')
        ? 'Global roles (all names)'
        : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`,
})

const decode = (data: `0x${string}`, abi: readonly unknown[]) =>
  decodeFunctionData({ abi: abi as never, data })

// ---------------------------------------------------------------------------

describe('portal resolver roles — scope collapse to ROOT', () => {
  it('HOP 1: a text-key-scoped grant produces a resource buildResourceToNameMap can never map', () => {
    const node = namehash('victim.eth')
    const textScoped = hex32(computeResolverResource(node, textPart('avatar')))
    const nameScoped = computeNameResource('victim.eth')

    const nodes: ResolverNode[] = [
      { id: 'n1', name: 'victim.eth', owner: null, resolver: null },
    ]
    const map = buildResourceToNameMap(nodes)

    console.log('name-scoped resource (part=0):', nameScoped)
    console.log('text-scoped resource (avatar):', textScoped)
    console.log('map knows name-scoped        :', map.get(nameScoped.toLowerCase()))
    console.log('map knows text-scoped        :', map.get(textScoped.toLowerCase()))

    expect(map.get(nameScoped.toLowerCase())).toBe('victim.eth')
    expect(map.get(textScoped.toLowerCase())).toBeUndefined()
  })

  it('HOP 2: the row for that delegate carries NO resolved name, so the sidebar shows no scope at all', () => {
    const textScoped = hex32(
      computeResolverResource(namehash('victim.eth'), textPart('avatar')),
    )
    const rows = groupRolesByAccount(
      [
        {
          account: DELEGATE,
          resource: textScoped,
          roleBitmap: hex32(ROLE_SET_TEXT_BIT),
        },
      ],
      buildResourceToNameMap([
        { id: 'n1', name: 'victim.eth', owner: null, resolver: null },
      ]),
    )

    const row = rows[0]!
    const scope = sidebarScope(row.resolvedNames)

    console.log('resolvedNames      :', row.resolvedNames)
    console.log('decodedRoles       :', row.decodedRoles)
    console.log('scope line rendered:', scope.scopeLineRendered)
    console.log('roleName (line 169):', JSON.stringify(scope.roleName))

    expect(row.resolvedNames).toEqual([])
    expect(scope.scopeLineRendered).toBe(false) // user is told NOTHING about scope
    expect(scope.roleName).toBe('') // == the ROOT sentinel
  })

  it('HOP 3+4: ticking one checkbox emits grantRootRoles — resolver-wide, not victim.eth', () => {
    const textScoped = hex32(
      computeResolverResource(namehash('victim.eth'), textPart('avatar')),
    )
    const row = groupRolesByAccount(
      [
        {
          account: DELEGATE,
          resource: textScoped,
          roleBitmap: hex32(ROLE_SET_TEXT_BIT),
        },
      ],
      buildResourceToNameMap([
        { id: 'n1', name: 'victim.eth', owner: null, resolver: null },
      ]),
    )[0]!

    const { roleName } = sidebarScope(row.resolvedNames)

    // The operator opens the row and ticks "Set Address" (manager).
    const edited = roleToPermissions(row.decodedRoles)
    edited.set('ROLE_SET_ADDR', { admin: false, manager: true })

    const { rolesToGrant } = computeRoleChanges(
      row.decodedRoles as string[],
      edited,
    )
    console.log('rolesToGrant       :', rolesToGrant)
    expect(rolesToGrant).toEqual(['ROLE_SET_ADDR'])

    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName, // <-- '' from line 169
      account: DELEGATE,
      roles: rolesToGrant,
      walletClient,
      chainId: CHAIN_ID,
    })

    const data = (intent.request as { data: `0x${string}` }).data
    const decoded = decode(data, [
      {
        name: 'grantRootRoles',
        type: 'function',
        stateMutability: 'nonpayable',
        inputs: [
          { name: 'roleBitmap', type: 'uint256' },
          { name: 'account', type: 'address' },
        ],
        outputs: [{ name: '', type: 'bool' }],
      },
    ])

    console.log('to                 :', (intent.request as { to: string }).to)
    console.log('raw calldata       :', data)
    console.log('decoded function   :', decoded.functionName)
    console.log('decoded args       :', decoded.args?.map(String))

    // grantRootRoles(uint256,address) selector
    expect(data.slice(0, 10)).toBe('0x072d5d77')
    expect(decoded.functionName).toBe('grantRootRoles')
    expect(String(decoded.args?.[0])).toBe(String(ROLE_SET_ADDR_BIT))
    expect(String(decoded.args?.[1]).toLowerCase()).toBe(DELEGATE.toLowerCase())

    // …and it is NOT the scoped call the operator believed they were making.
    const scopedIntent = prepareGrantResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: 'victim.eth',
      account: DELEGATE,
      roles: rolesToGrant,
      walletClient,
      chainId: CHAIN_ID,
    })
    const scopedData = (scopedIntent.request as { data: `0x${string}` }).data
    console.log('scoped calldata    :', scopedData)
    expect(scopedData).not.toBe(data)
    expect(scopedData.slice(0, 10)).not.toBe(data.slice(0, 10))
  })

  it('same collapse with NO exotic scope: a node simply missing from the indexer `nodes` page', () => {
    // `getResolverOverview` asks for `nodes { ... }` with no pagination args
    // (useResolverOverview.ts:75-85); any node the indexer does not return in
    // that page is unmappable.
    const rows = groupRolesByAccount(
      [
        {
          account: DELEGATE,
          resource: computeNameResource('victim.eth'),
          roleBitmap: hex32(ROLE_SET_ADDR_BIT),
        },
      ],
      buildResourceToNameMap([
        // victim.eth is NOT in this page
        { id: 'n2', name: 'other.eth', owner: null, resolver: null },
      ]),
    )
    const scope = sidebarScope(rows[0]!.resolvedNames)
    console.log('resolvedNames      :', rows[0]!.resolvedNames)
    console.log('roleName           :', JSON.stringify(scope.roleName))
    expect(scope.roleName).toBe('')

    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: scope.roleName,
      account: DELEGATE,
      roles: ['ROLE_SET_ADDR'] as ResolverRoleKey[],
      walletClient,
      chainId: CHAIN_ID,
    })
    const data = (intent.request as { data: `0x${string}` }).data
    const decoded = decode(data, [
      {
        name: 'revokeRootRoles',
        type: 'function',
        stateMutability: 'nonpayable',
        inputs: [
          { name: 'roleBitmap', type: 'uint256' },
          { name: 'account', type: 'address' },
        ],
        outputs: [{ name: '', type: 'bool' }],
      },
    ])
    console.log('decoded function   :', decoded.functionName, decoded.args?.map(String))
    expect(decoded.functionName).toBe('revokeRootRoles')
  })

  it('CONTROL: a plain single-name row encodes the scoped authorizeNameRoles call', () => {
    const rows = groupRolesByAccount(
      [
        {
          account: DELEGATE,
          resource: computeNameResource('victim.eth'),
          roleBitmap: hex32(ROLE_SET_ADDR_BIT),
        },
      ],
      buildResourceToNameMap([
        { id: 'n1', name: 'victim.eth', owner: null, resolver: null },
      ]),
    )
    const scope = sidebarScope(rows[0]!.resolvedNames)
    expect(scope.roleName).toBe('victim.eth')
    expect(scope.scopeLineText).toBe('Roles scoped to victim.eth')

    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: scope.roleName,
      account: DELEGATE,
      roles: ['ROLE_SET_TEXT'],
      walletClient,
      chainId: CHAIN_ID,
    })
    const data = (intent.request as { data: `0x${string}` }).data
    const decoded = decode(data, [
      {
        name: 'authorizeNameRoles',
        type: 'function',
        stateMutability: 'nonpayable',
        inputs: [
          { name: 'name', type: 'bytes' },
          { name: 'roleBitmap', type: 'uint256' },
          { name: 'account', type: 'address' },
          { name: 'grant', type: 'bool' },
        ],
        outputs: [{ name: '', type: 'bool' }],
      },
    ])
    console.log('control function   :', decoded.functionName)
    console.log('control args       :', decoded.args?.map(String))
    expect(decoded.functionName).toBe('authorizeNameRoles')
  })
})

describe('portal resolver roles — wrong-name target when one account holds two names', () => {
  it('an edit the operator makes while reading "b.eth" is encoded against a.eth', () => {
    const rows = groupRolesByAccount(
      [
        {
          account: DELEGATE,
          resource: computeNameResource('a.eth'),
          roleBitmap: hex32(ROLE_SET_TEXT_BIT),
        },
        {
          account: DELEGATE,
          resource: computeNameResource('b.eth'),
          roleBitmap: hex32(ROLE_SET_TEXT_BIT),
        },
      ],
      buildResourceToNameMap([
        { id: 'n1', name: 'a.eth', owner: null, resolver: null },
        { id: 'n2', name: 'b.eth', owner: null, resolver: null },
      ]),
    )
    const row = rows[0]!
    const scope = sidebarScope(row.resolvedNames)
    console.log('resolvedNames      :', row.resolvedNames)
    console.log('UI scope line      :', scope.scopeLineText)
    console.log('roleName used      :', JSON.stringify(scope.roleName))

    // The UI lists BOTH names as the scope…
    expect(scope.scopeLineText).toBe('Roles scoped to a.eth, b.eth')
    // …but every write targets only the first.
    expect(scope.roleName).toBe('a.eth')

    const edited = roleToPermissions(row.decodedRoles)
    edited.set('ROLE_SET_ADDR', { admin: false, manager: true })
    const { rolesToGrant } = computeRoleChanges(
      row.decodedRoles as string[],
      edited,
    )

    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: scope.roleName,
      account: DELEGATE,
      roles: rolesToGrant,
      walletClient,
      chainId: CHAIN_ID,
    })
    const data = (intent.request as { data: `0x${string}` }).data
    const decoded = decode(data, [
      {
        name: 'authorizeNameRoles',
        type: 'function',
        stateMutability: 'nonpayable',
        inputs: [
          { name: 'name', type: 'bytes' },
          { name: 'roleBitmap', type: 'uint256' },
          { name: 'account', type: 'address' },
          { name: 'grant', type: 'bool' },
        ],
        outputs: [{ name: '', type: 'bool' }],
      },
    ])
    console.log('decoded args       :', decoded.args?.map(String))
    // dnsEncode('a.eth') = 0x01 61 03 657468 00 — never b.eth.
    expect(String(decoded.args?.[0])).toBe('0x01610365746800')
  })
})

describe('portal resolver roles — the inverse: UI says GLOBAL, calldata is scoped', () => {
  it('"Remove user" on a row that holds root + per-name roles leaves the GLOBAL grant intact', () => {
    const rows = groupRolesByAccount(
      [
        // indexer returns the root grant first…
        {
          account: DELEGATE,
          resource: ROOT_RESOURCE,
          roleBitmap: hex32(ROLE_SET_ADDR_BIT),
        },
        // …and a per-name grant second
        {
          account: DELEGATE,
          resource: computeNameResource('alice.eth'),
          roleBitmap: hex32(ROLE_SET_TEXT_BIT),
        },
      ],
      buildResourceToNameMap([
        { id: 'n1', name: 'alice.eth', owner: null, resolver: null },
      ]),
    )

    const row = rows[0]!
    const scope = sidebarScope(row.resolvedNames)

    console.log('resolvedNames      :', row.resolvedNames)
    console.log('decodedRoles UNION :', row.decodedRoles)
    console.log('UI scope line      :', scope.scopeLineText)
    console.log('roleName used      :', JSON.stringify(scope.roleName))

    // One row, two resources merged.
    expect(row.roles).toHaveLength(2)
    expect(row.resolvedNames).toEqual(['(root)', 'alice.eth'])
    // The banner the operator reads:
    expect(scope.scopeLineText).toBe('Global roles (all names)')
    // …but the name the write uses:
    expect(scope.roleName).toBe('alice.eth')

    // handleRemoveUser -> revoke `decodedRoles` (the union) at `roleName`.
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: scope.roleName,
      account: DELEGATE,
      roles: row.decodedRoles as ResolverRoleKey[],
      walletClient,
      chainId: CHAIN_ID,
    })
    const data = (intent.request as { data: `0x${string}` }).data
    const decoded = decode(data, [
      {
        name: 'authorizeNameRoles',
        type: 'function',
        stateMutability: 'nonpayable',
        inputs: [
          { name: 'name', type: 'bytes' },
          { name: 'roleBitmap', type: 'uint256' },
          { name: 'account', type: 'address' },
          { name: 'grant', type: 'bool' },
        ],
        outputs: [{ name: '', type: 'bool' }],
      },
    ])

    console.log('raw calldata       :', data)
    console.log('decoded function   :', decoded.functionName)
    console.log('decoded args       :', decoded.args?.map(String))

    expect(decoded.functionName).toBe('authorizeNameRoles')
    // dnsEncode('alice.eth') — the revoke lands on alice.eth only.
    expect(String(decoded.args?.[0])).toContain('05616c696365')
    expect(String(decoded.args?.[3])).toBe('false')
    // The root grant is never touched: no revokeRootRoles anywhere in the tx.
    expect(data.slice(0, 10)).not.toBe('0xce156e82')
  })
})
```

### Complete PoC source — `role-set-data-invisible.poc.test.ts`

```ts
/**
 * HERMETIC PoC — the portal's LOCAL resolver-role table is missing
 * `ROLE_SET_DATA` (nybble 9, `1n << 36n`), which ensjs — the ENS team's own
 * client for this exact contract — does define.
 *
 * Consequence chain (all real repo / node_modules code, no network, no chain):
 *
 *   apps/portal/src/lib/roles/resolverRoles.ts:24-45   `resolverRoles` table
 *      omits ROLE_SET_DATA / ROLE_SET_DATA_ADMIN.
 *   apps/portal/src/lib/roles/resolverRoles.ts:113-127 `decodeResolverRoleBitmap`
 *      iterates that table, so bit 36 is DROPPED from the decoded role list.
 *   apps/portal/src/lib/roles/resolverRoles.ts:196      `groupRolesByAccount`
 *      stores the lossy decode as `decodedRoles`.
 *   apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:167
 *      `decodedRoles = row.original.decodedRoles`
 *   apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:317-321
 *      "Remove user" sends EXACTLY `roles: decodedRoles`.
 *   apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-60
 *      -> ensjs `revokeResolverRolesWriteParameters`
 *      -> `encodeResolverRoleBitmap(roles)`  (ensjs, full table)
 *
 * So the revoke bitmap the owner signs cannot contain bit 36. An account that
 * holds ROLE_SET_DATA keeps it after "Remove user" reports success, and the
 * roles table never showed it in the first place.
 *
 * `ROLE_SET_DATA` authorises `setData` on a PermissionedResolver — writing raw
 * resolver storage for the resource, i.e. the addr/text/contenthash a name
 * resolves to.
 */
import {
  encodeResolverRoleBitmap,
  RESOLVER_ROLE_SET_ADDR,
  RESOLVER_ROLE_SET_DATA,
  RESOLVER_ROLE_SET_TEXT,
} from '@ensdomains/ensjs/public/v2'
import { describe, expect, it } from 'vitest'

import {
  decodeResolverRoleBitmap,
  groupRolesByAccount,
  resolverPermissions,
  resolverRoles,
} from '@/lib/roles/resolverRoles'

const DELEGATE = '0xdddddddddddddddddddddddddddddddddddddddd'
const RESOURCE = `0x${'11'.repeat(32)}`

describe('portal resolver roles — ROLE_SET_DATA is invisible and unrevokable', () => {
  it('HOP 1: ensjs defines ROLE_SET_DATA at bit 36; the portal table does not', () => {
    expect(RESOLVER_ROLE_SET_DATA).toBe(1n << 36n)
    // The portal's own table has no entry with that value, under any key.
    const portalValues = Object.values(resolverRoles)
    console.log('ensjs RESOLVER_ROLE_SET_DATA      :', RESOLVER_ROLE_SET_DATA)
    console.log(
      'portal table has that bit?       :',
      portalValues.includes(RESOLVER_ROLE_SET_DATA),
    )
    console.log(
      'portal table keys               :',
      Object.keys(resolverRoles).filter((k) => !k.endsWith('_ADMIN')),
    )
    expect(portalValues).not.toContain(RESOLVER_ROLE_SET_DATA)
    expect(resolverPermissions.map((p) => p.key)).not.toContain('ROLE_SET_DATA')
  })

  it('HOP 2: a bitmap carrying SET_ADDR + SET_DATA decodes to SET_ADDR only', () => {
    const onChainBitmap = RESOLVER_ROLE_SET_ADDR | RESOLVER_ROLE_SET_DATA
    const decoded = decodeResolverRoleBitmap(onChainBitmap)
    console.log('on-chain bitmap                 :', onChainBitmap)
    console.log('portal decodeResolverRoleBitmap :', decoded)
    expect(decoded).toEqual(['ROLE_SET_ADDR'])
    expect(decoded).not.toContain('ROLE_SET_DATA')
  })

  it('HOP 3: groupRolesByAccount propagates the lossy decode into the table row', () => {
    const [group] = groupRolesByAccount([
      {
        account: DELEGATE,
        resource: RESOURCE,
        roleBitmap: (
          RESOLVER_ROLE_SET_ADDR | RESOLVER_ROLE_SET_DATA
        ).toString(),
      },
    ])
    console.log('row.decodedRoles                :', group.decodedRoles)
    expect(group.decodedRoles).toEqual(['ROLE_SET_ADDR'])
  })

  it('HOP 4: the revoke bitmap "Remove user" signs cannot clear bit 36', () => {
    const onChainBitmap = RESOLVER_ROLE_SET_ADDR | RESOLVER_ROLE_SET_DATA
    const [group] = groupRolesByAccount([
      { account: DELEGATE, resource: RESOURCE, roleBitmap: onChainBitmap.toString() },
    ])

    // ResolverRolesSidebar.tsx:317-321 passes `roles: decodedRoles` verbatim.
    // revokeResolverRoles.ts -> ensjs encodeResolverRoleBitmap.
    const revokeBitmap = encodeResolverRoleBitmap(
      group.decodedRoles as Parameters<typeof encodeResolverRoleBitmap>[0],
    )
    const residual = onChainBitmap & ~revokeBitmap

    console.log('bitmap held on-chain            :', onChainBitmap)
    console.log('bitmap "Remove user" revokes    :', revokeBitmap)
    console.log('residual AFTER a "successful"   :', residual)
    console.log(
      'residual === ROLE_SET_DATA?     :',
      residual === RESOLVER_ROLE_SET_DATA,
    )

    expect(revokeBitmap).toBe(RESOLVER_ROLE_SET_ADDR)
    expect(residual).toBe(RESOLVER_ROLE_SET_DATA)
    expect(residual).not.toBe(0n)
  })

  it('CONTROL: a bitmap with only table-known roles round-trips losslessly', () => {
    const clean = RESOLVER_ROLE_SET_ADDR | RESOLVER_ROLE_SET_TEXT
    const [group] = groupRolesByAccount([
      { account: DELEGATE, resource: RESOURCE, roleBitmap: clean.toString() },
    ])
    const revokeBitmap = encodeResolverRoleBitmap(
      group.decodedRoles as Parameters<typeof encodeResolverRoleBitmap>[0],
    )
    console.log('control held                    :', clean)
    console.log('control revoked                 :', revokeBitmap)
    console.log('control residual                :', clean & ~revokeBitmap)
    expect(clean & ~revokeBitmap).toBe(0n)
  })
})
```
