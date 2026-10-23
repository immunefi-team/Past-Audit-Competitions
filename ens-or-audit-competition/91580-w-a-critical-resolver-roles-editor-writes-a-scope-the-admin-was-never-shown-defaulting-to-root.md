# 91580 w a critical resolver roles editor writes a scope the admin was never shown defaulting to root

**Submitted on Sep 2nd 2026 at 19:38:18 UTC by @ChaseTheLight for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91580
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions
    * Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated)
    * Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Explorer's permissioned-resolver roles page merges every role an account holds into one table row, drops the resource from everything the editor reads, and then reconstructs a scope for the admin's edit from what survived. It does that twice, with two rules that read the same array and disagree: one produces the label the admin reads, the other produces the scope written on chain.

The write rule falls back to the empty string, which does not narrow the call: it selects a different function on the resolver. So an admin who opens a row that displays no scope, ticks one permission and presses Save, issues `grantRootRoles`, handing that account the permission over every name the resolver serves.

The grant lands whenever the admin holds the admin bit for the role being granted, which every resolver this application deploys gives its owner. On a real `PermissionedResolver`, the escalated account then wrote the address record of a name it was never scoped to, and the resolver returns its address for that name. The PoC creates that row; it does not survey deployed resolvers for one.

The same collapse also defeats removal, on a row this application builds unaided: give one account a root grant and a name grant, both of which the Add User sheet issues routinely, and "Remove user" revokes at the name only.

The two variants are not the same defect twice:

* The blank row **creates** root authority the admin never chose to give.
* The root-plus-name row **preserves** root authority the admin did once give and now believes they have withdrawn, behind a dialog saying the user was removed from all roles.

Both are measured on chain.

## Vulnerability Details

### Root cause

A permissioned resolver holds roles per resource:

* `ROOT_RESOURCE`, meaning every name the resolver serves
* A single name's resource
* A record-scoped resource, one record on one name

The roles page merges all of them into one row per account, then has to guess a scope back out of the merged row when an admin saves an edit.

The label the admin sees comes from one rule (`ResolverRolesSidebar.tsx:391-397`). The scope written on chain comes from another (`:169`). Both read the same `resolvedNames` array:

| `resolvedNames` on the row | Label shown                        | Scope written |
| -------------------------- | ---------------------------------- | ------------- |
| `['(root)']`               | Global roles (all names)           | root          |
| `['alice.eth']`            | Roles scoped to alice.eth          | alice.eth     |
| `['(root)', 'alice.eth']`  | Global roles (all names)           | **alice.eth** |
| `['alice.eth', 'bob.eth']` | Roles scoped to alice.eth, bob.eth | **alice.eth** |
| `[]`                       | **no scope line renders at all**   | **root**      |

Every row in that table is measured, not asserted, on at least one column: the Label column for all five shapes and the Scope written column for four of them, including every shape this report turns on.

The last row is the escalation, and the Critical rests on it. The root-plus-name row fails the other way: the admin is told the roles are global, edits them, and the edit lands on one name.

The information the write needs is discarded during grouping and then guessed at. `groupRolesByAccount` keys on the account alone (`apps/portal/src/lib/roles/resolverRoles.ts:179-221`). The per-role `resource` is read, used for one display lookup, and then dropped. The merge is deliberate: the project's own test asserts it (`resolverRoles.test.ts:185-197`, "should merge the same account across resources").

What follows from it is not. `decodedRoles` is a flat `string[]`, so a ticked permission cannot be attributed to the role that carried it. The raw roles do survive on the group (`AccountRoleGroup.roles`, `:168-173`, each still carrying its `resource`); the sidebar never reads them.

The name lookup only knows about nodes the resolver query returned: `buildResourceToNameMap` seeds `ROOT_RESOURCE` as `(root)` and then adds `computeNameResource(node.name)` for each returned node that has a name (`resolverRoles.ts:147-160`). Anything else resolves to `null` and is dropped from `resolvedNames`, so an unrecognised resource is indistinguishable from no resource.

The sidebar then picks a scope out of what survived (`ResolverRolesSidebar.tsx:169`):

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

It renders a label out of the same array by a different rule (`:391-397`):

```tsx
{selectedAccount && resolvedNames.length > 0 && (
  <p className="text-sm text-muted-foreground">
    {resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
  </p>
)}
```

`roleName` is the only scope handed to both mutations, for save (`:299`) and for remove (`:318`), and both writers read the empty string as root. `grantResolverRoles.ts:47-60`, with the same branch in `revokeResolverRoles.ts:47-60`:

```ts
name === ''
  ? {
      resolverAddress,
      targetAccount: account,
      scope: 'root',
      roles,
    }
  : {
      resolverAddress,
      targetAccount: account,
      scope: 'name',
      name,
      roles,
    },
```

That meaning is intended and documented at the parameter itself (`grantResolverRoles.ts:20-21`):

```ts
/** Dotted name (e.g. "myname.eth") or empty string for ROOT_RESOURCE (all names). */
readonly name: string
```

The behaviour is also observable. Decoding what the builder produces for each branch gives two different functions on the resolver:

```
[poc] empty scope : 0x072d5d77 grantRootRoles
[poc] name scope  : 0xbbd9abb5 authorizeNameRoles
```

Root calldata does not decode under the name-scoped ABI. The writers honour that contract. The defect is that the sidebar reaches it by falling through a `??`, so a failed lookup becomes the most privileged option available.

Nothing downstream restores the scope. The same `name` string is carried unchanged from `handleSaveChanges` (`:295-305`) through the modal's `onStart` (`:486-502`) to the mutation (`:195-249`) and on to `grantResolverRoles`, with no normalisation, re-resolution or validation step anywhere on it.

The roles editor's confirmation carries no scope at all: `transactionName` is `Update roles for ${pendingAction?.account ?? ''}` and the title is "Save resolver role changes" (`ResolverRolesSidebar.tsx:339-349`). At no point is the admin shown what they are authorising.

The builder does compose a `description` that names the scope, "Grant resolver roles for" followed by the name or `(root)` (`grantResolverRoles.ts:111`), but nothing renders it: the transaction manager displays only `title` and `transactionName` (`TransactionInfoContent.tsx:35,43`, `TransactionStateContent.tsx:59,113`). The sibling grant screen puts the scope where it is read, in `transactionName` (`ResolverAddUserSheet.tsx:321`). The roles editor is the screen that leaves it out.

"Remove user" fails the same way, and on the root-plus-name row it fails in the direction that matters. It sends the merged permission list with the single guessed scope (`:318`), so a row carrying both a root grant and a name grant produces `authorizeNameRoles(dnsEncode(name), bitmap, account, false)` and never touches `revokeRootRoles`; that behaviour is measured.

The two grants sit at different resources, so a revoke aimed at the name resource does not reach the root one, which is the same resource model the grant path turns on. The dialog the admin confirms first reads:

> Are you sure you want to remove this user from all roles? This action cannot be undone.

That row needs nothing unusual to exist: the same page creates it, a root grant once and a name grant later. The consequence is measured on chain. After the revoke that row produces is mined, `hasRootRoles` for the grantee still reads `true`: the account the admin was told had been removed from all roles keeps `ROLE_SET_ADDR` at `ROOT_RESOURCE`, over every name the resolver serves.

The grantee ends up holding exactly what the escalation confers, reached with two grants the Explorer's own Add User sheet issues routinely and no resource this UI cannot create. The route differs, and it matters separately: in the escalation the application creates authority the admin never chose to give, while here it preserves authority the admin gave once and believes they have withdrawn.

### Trigger and preconditions

The actor is an admin of a permissioned resolver, performing a routine permission change on the roles page. No attacker is present at the time.

For the escalation case the editing admin must hold root-level grant authority on that resolver; otherwise the root grant reverts and the outcome is a confusing failure rather than an escalation. The roles page only enables editing for an account that already holds a root-scoped role on the resolver: `canManageRoles` is `Boolean(hasRootRole)`, which comes from the root-roles overload (`routes/resolver/$address/roles/index.tsx:41-50`, `useHasRoles.ts:30-34`).

So the person reaching this write is already a root-level principal on that resolver. Holding a root manager role is not the same as holding the admin bit for the role being granted. For every resolver this application deploys, the two arrive together.

`getResolverInitCalldata` initialises each new `PermissionedResolver` with `initialize(owner, permissionedResolverRoleBitmap, [])`, and that bitmap is `0x1111...1111` (`permissionedResolver.ts:21,39-43`), so every fourth bit is set. `ROLE_SET_ADDR_ADMIN` is `(1n << 0n) << 128n` (`resolverRoles.ts:26`), and bit 128 is among them, so the deploying owner holds the admin bit by construction.

The on-chain test measures the same outcome from the other direction: the `grantRootRoles` transaction mines and the root grant lands. An account handed a root manager role after deployment need not hold the admin bit, and for that account the write fails rather than escalating.

Three ways a row reaches the empty state, the second of which is the one executed on chain.

{% stepper %}
{% step %}

### A returned node that has no name

The map skips any node whose `name` is falsy (`resolverRoles.ts:154`). That guard is the app's own acknowledgement that the case occurs, and the generated schema agrees: `Domain.name` is nullable (`packages/indexer/graphql.gen.ts:102`), while the app's own `ResolverNode` type declares `name: string` and does not model the null (`useResolverOverview.ts:35`).

A role held on such a node is returned in `roles` and has no entry in the map, so it resolves to nothing. This route assumes only that the indexer returns a node whose name its own schema already declares nullable, and needs no assumption about pagination.
{% endstep %}

{% step %}

### A resource this UI never created

A role written directly against the resolver contract, or by an earlier tool. `buildResourceToNameMap` only ever holds `ROOT_RESOURCE` plus `resource(node, 0)` for the returned nodes, so a role granted at a text-scoped or coin-scoped resource, which ensjs supports and the indexer models with `coinType` and `recordKey` (`graphql.gen.ts:381-390`), can never be named by it.

The on-chain test takes this route, and measures it: a part-scoped `authorizeAddrRoles` grant landed on resource `52615576...777586`, while `computeNameResource` for the same name produces `21722063...996533`. The real `groupRolesByAccount` returned `resolvedNames: []` for that resource against a map built from both names in the node list.
{% endstep %}

{% step %}

### A node list truncated relative to the role list

The generated schema declares `ResolverDetail.nodes` as accepting a `first` limit (`graphql.gen.ts:940-942`) while `ResolverDetail.roles` accepts none, and the Explorer requests `nodes` without `first` (`useResolverOverview.ts:75`). Whether such a default is applied is the indexer's behaviour and is not readable from this repository, which is why it is listed last.
{% endstep %}
{% endstepper %}

The multi-name and root-plus-name rows need nothing unusual: the same page creates them. `ResolverAddUserSheet` writes at root when the node combobox is left at its default (`ROOT_NODE_VALUE = ''`, `:39,127,205`) and at a name when one is picked, so two name grants produce the multi-name row, and a root grant once with a name grant later produces the root-plus-name row.

The step-by-step sequence, and which steps are measured in which PoC file, is in the Proof of Concept section.

### Recommended fix

The structural fix is to make one row mean one scope. Key `groupRolesByAccount` on `(account, resource)` rather than on the account alone, and carry the resource through `AccountRoleGroup` so each grant and revoke is issued against the resource it was read from.

The data is already there twice over. `RoleInput` need only widen to include the per-role `name` the indexer already returns and the app discards (`useResolverOverview.ts:14-22`). That field is nullable, so the cleaner source is `ResolverDetail.namedResources` (`graphql.gen.ts:927`, entry type at `:381-390`), which carries `resource`, `name`, `coinType`, `recordKey` and `recordKind`, and so covers exactly the record-scoped resources `buildResourceToNameMap` cannot name. The portal never asks for it.

Until then, the minimal fail-closed change is for the sidebar to refuse to write when the row's scope is ambiguous or unresolvable, instead of falling through `??` to the most privileged option:

```ts
const isScopeWritable = resolvedNames.length === 1
const roleName = resolvedNames[0] === '(root)' ? '' : (resolvedNames[0] ?? '')
```

Add `isScopeWritable` to the guards in `handleSaveChanges` and `handleRemoveUser`. Only a row that resolved to exactly one entry can be written from this screen; the empty, two-name and root-plus-name shapes are all blocked.

The full patch and its measured effect are in the Proof of Concept section: with it applied, the four wrong-scope writes stop being issued and the one legitimate write is unaffected.

Also give the writers a discriminated scope, `{ kind: 'root' } | { kind: 'name'; name: string }`, instead of a `string` whose empty value is the most privileged one, so that any caller failing to resolve a scope cannot widen it by default.

## Impact Details

### The in-scope impact this maps to

The governing entry, quoted verbatim from the programme's Impacts in Scope list:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

This report files at **Critical** on that entry. The application displays one scope and pre-fills a different one into the transaction the admin approves:

* On the root-plus-name and two-name rows the label reads "Global roles (all names)" while the write goes to a single name.
* On the blank row no scope is displayed at all while the write goes to root.

The admin submits a transaction whose effect is not the one shown. The empty string is the sharpest case, because it does not narrow `authorizeNameRoles` but selects `grantRootRoles`, a different function on the resolver.

Every step of that is executed. The value on the wire is measured from the Save button (`expect(wire.name).toBe('')`), the calldata it produces is decoded to `grantRootRoles`, and the transaction mines on a real `PermissionedResolver`, after which `hasRootRoles` for the grantee reads `true` where it read `false` before.

A second entry on the same list reaches this finding and carries the same rung:

> Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

The roles page only enables editing for a connected admin wallet, so the wallet is connected by construction, and the first item in that list describes what happens to the scope parameter. It is cited second because its head clause can be read as requiring an adversary operating at the wallet boundary, and there is none here: the application's own `??` supplies the wrong parameter.

The finding does not depend on that reading, and both entries are Critical.

### Loss mechanism and its bound

`ROLE_SET_ADDR` at `ROOT_RESOURCE` is "Can set address records" in the app's own permission list (`resolverRoles.ts:58-62`), and at root scope ensjs documents the grant as "globally (any name, any record type)" (`@ensdomains/ensjs`, `src/actions/wallet/v2/grantResolverRoles.ts:46` and `:176`; that is a pinned dependency rather than part of the export, so it is not openable from the snapshot alone).

The holder can rewrite the address record of every name that resolver serves, repeatedly, with no further interaction from the admin. Which names those are is a deployment property; the PoC measures the authority, not a census of what sits behind any particular resolver.

The loss path is misdirected payment. An ENS name is a payment destination, and the grantee can rewrite the address record of the names the compromised resolver serves. A client that resolves one of those names through that resolver receives the grantee's address, and the resolution is authoritative, so nothing in the sending path has grounds to reject it: the record was written by an account the resolver authorises.

The Explorer is itself one of those clients, which keeps the last hop inside the listed asset rather than resting on third-party behaviour. `useAddressResolution` is documented as shared by every name-or-address input in the app, naming the transfer recipient and the add-user sheets (`features/address/hooks/useAddressResolution.ts:43-45`).

It reaches `getResolvedAddress`, whose docblock states that names go through the universal resolver (`features/address/queries/getResolvedAddress.ts:14-16`), which delegates to `resolveAddressOrName` (same file, `:26`), defined at `features/roles/helpers/addUser.handlers.ts:11`, and that is what calls `getEnsAddress(client, { name, universalResolverAddress })` (`addUser.handlers.ts:20-23`).

The hook's consumers include the send-name recipient field (`features/transfer/components/SendNameForm.tsx`) and all three Add User sheets. So the impact entry's own sub-bullet, resolving or rendering an ENS name to an attacker-controlled address, is satisfied on the Explorer's own input path and not only in an external wallet.

What the PoC measures is the write and the read back from the resolver. It performs no payment and does not bind its test name in the registry, so the last hop, from a rewritten record to a user's transfer, follows from how ENS resolution works rather than from anything executed here.

The size of the loss is bounded by the value routed, during the window, to every name that resolver serves, and the window is wide because the grant is silent on both sides: the admin saw no scope on the screen and none in the confirmation modal, and the grantee is not required to act immediately. The grant is on chain the moment it confirms.

A specific figure is not offered because it is not derivable from the audited assets: the report does not survey deployed resolvers, and the number of names behind any given permissioned resolver is a deployment property rather than a code property. The code establishes the multiplier: one grant covers every name on the resolver. The PoC shows that multiplier holding on a deployed contract.

### Why not lower, and the out-of-scope entries to weigh

No lower entry on the list covers an authority grant landing at a scope the admin did not choose, and clearing such a grant is not guaranteed by a single action.

**"Impacts caused by attacks requiring access to privileged addresses (governance, strategist) except in such cases where the contracts are intended to have no privileged access to functions that make the attack possible"** does not apply. The exclusion turns on an attacker needing privileged access. The account that gains authority here holds one narrow role and needs no privilege at any point. The privileged party is the victim, an admin performing a routine edit, and the privilege is what the application spends on their behalf without telling them.

**"UX and UI impacts that do not materially disrupt use of the platform"** does not apply either. The defect site is a React component, so the entry is worth naming, but the outcome is not a rendering artefact: the transaction mines, `hasRootRoles` for the grantee reads `false` before it and `true` after, and a `setAddr` that reverts on a clean resolver then succeeds against the escalated one.

The reason a reviewer might still land lower is the precondition set on the escalation case: it needs an admin who holds root grant authority, and a row in the empty state. Of the three routes to that row, the one the PoC executes is a part-scoped grant, which the Explorer's own screens cannot issue; the other two rest on indexer behaviour not readable from this repository.

The empty row is therefore shown reachable structurally and in the harness, not surveyed in production, and that is this report's softest point. The root-plus-name row carries no such dependency: the same page creates it in two ordinary grants, and it misroutes both the save and the remove.

### Not a duplicate, and scope

Nothing in the Known Issues document carries this root cause. The nearest entry by vocabulary is **QA-01** ("Name transfers with certain role combinations break follow-up actions"), listed against both apps.

It is a different flow: transferring a name, then hitting errors when updating profile records or setting a primary name. Its stated symptom is an error, and it says nothing about resolver role scopes, `ROOT_RESOURCE`, or the roles editor.

Part 2 also states, at its head, where the line falls for anything in that neighbourhood:

> However, a demonstrated security consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding.

Even on the most generous reading of QA-01's scope, this report is what that sentence carves in: a silent grant of authority that should not have been given, not an error.

Its tracked instances **WEB-1257** (a registry not detached after a parent transfer), **WEB-1305** ("Transferring roles to a V1 name hangs") and **WEB-716** ("No way to replace an existing subregistry") are a registry-detachment bug, a pending-transaction defect and a missing action respectively; none touches the roles grouping or the scope written.

Nothing in Part 1 or Part 2 mentions `groupRolesByAccount`, `resolvedNames`, `ROOT_RESOURCE`, or the resolver roles sidebar.

The defect is in `apps/portal`, a listed asset, in the app's own grouping helper and component. The out-of-scope list places "The ENS smart contracts, subgraph/indexer, and NFT metadata service" outside the programme; none of them is at fault here. The indexer supplies a per-role `name` (`useResolverOverview.ts:21,97`) and the app discards it. There is no feature flag and no branch that avoids the defect: the fallback is a `??` on the expression itself.

## References

Repository: `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal` (the Explorer asset, dated 6 August 2026). All paths below are relative to the repository root.

1. `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx` - `#L166-L169` (the write rule, the defect), `#L299` (save), `#L318` (remove), `#L339-L349` (the confirmation copy, which carries no scope), `#L391-L397` (the display rule)
2. `apps/portal/src/lib/roles/resolverRoles.ts` - `#L147-L160` (`buildResourceToNameMap`), `#L162-L166` (`RoleInput`, which drops the name), `#L168-L173` (`AccountRoleGroup`, which keeps each `resource` unread), `#L179-L221` (`groupRolesByAccount`)
3. `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts` - `#L20-L21` (the parameter doc: empty string means ROOT\_RESOURCE), `#L47-L60` (the root/name branch); `revokeResolverRoles.ts` - `#L47-L60` (the same branch)
4. `apps/portal/src/features/resolver/hooks/useResolverOverview.ts` - `#L14-L22` (`ResolverRole` carries `name`), `#L35` (`ResolverNode` declares `name: string`, not modelling the null)
5. `apps/portal/src/routes/resolver/$address/roles/index.tsx` - `#L41-L50` (the root-roles gate); `apps/portal/src/features/registry/hooks/useHasRoles.ts` - `#L30-L34` (the root-roles overload)
6. `packages/indexer/graphql.gen.ts` - `#L102` (nullable `Domain.name`), `#L381-L390` (`NamedResourceEntry` carries `resource`, `name`, `coinType`, `recordKey`, `recordKind`), `#L927` (`ResolverDetail.namedResources`, the field the portal never queries), `#L940-L942` (`nodes` accepts `first`, `roles` does not)
7. `apps/portal/src/features/resolver/utils/permissionedResolver.ts` - `#L21` (the `0x1111...1111` role bitmap), `#L39-L43` (`initialize(owner, bitmap, [])` for every resolver the app deploys); `apps/portal/src/lib/roles/resolverRoles.ts` - `#L26` (`ROLE_SET_ADDR_ADMIN` is bit 128)
8. `apps/portal/src/features/transaction-manager/components/TransactionInfoContent.tsx` - `#L35`, `#L43`; `TransactionStateContent.tsx` - `#L59`, `#L113` (the only transaction fields rendered are `title` and `transactionName`; `description` is never displayed); `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx` - `#L321` (the sibling screen that does put the scope in `transactionName`)
9. The Explorer's own resolution path, which consumes the rewritten record: `apps/portal/src/features/address/hooks/useAddressResolution.ts` - `#L43-L45` (shared by every name-or-address input); `apps/portal/src/features/address/queries/getResolvedAddress.ts` - `#L14-L16` (names go through the universal resolver); `apps/portal/src/features/roles/helpers/addUser.handlers.ts` - `#L11`, `#L20-L23` (`getEnsAddress` against `universalResolverAddress`); consumers at `apps/portal/src/features/transfer/components/SendNameForm.tsx` and the three Add User sheets

File fingerprints at the audited snapshot:

| File                                                                    | Bytes  | Lines | sha256                                                             |
| ----------------------------------------------------------------------- | ------ | ----- | ------------------------------------------------------------------ |
| `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx` | 16,429 | 515   | `ac954b1790e50f45068852e55c5bac6ba61d6dedd30c2e84f29ea925642b825a` |
| `apps/portal/src/lib/roles/resolverRoles.ts`                            | 5,773  | 221   | `45e347ba9300ec3605b4bebdd7ded7e265a828a39fd18e87d5c763cd781faab7` |
| `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts`       | 2,956  | 119   | `8cad669da9d57b366622ac1ce17340b8b86e19ff06d77faa2f5670f7ee94a926` |
| `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts`      | 3,008  | 118   | `01a59e8853972fceb876bc573d8b1ff91fb693622c2975b9f6e667fbbc41f771` |

## Link to Proof of Concept

<https://gist.github.com/ChaseTheLight01/4c953fcaa1531833f05517b146804444>
