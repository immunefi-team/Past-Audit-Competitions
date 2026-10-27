# 92174 w a critical registration and migration grant resolver global not name scoped root roles that detach the resolver never revokes so a name s seller keeps redirecting the buyer s addr 6&#x20;

**Submitted on Sep 7th 2026 at 20:32:36 UTC by @velikoros for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92174
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/smart-account>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

### Brief/Intro

Every time `packages/smart-account` registers a name for a smart-account wallet (HCA), it deploys the wallet a `PermissionedResolver` and grants the wallet `ROLES_ALL` **at the resolver's root resource** (`authorizeNameRoles(hex"00", ROLES_ALL, wallet, true)`) — not scoped to the specific name being registered. `apps/manager`'s migration flow makes the identical root grant for every co-admin wallet it adds. Root-scoped roles on this resolver are not per-name: they authorize the holder to write **any** record on **any** node whose registry entry happens to point at that resolver, including names the holder never registered. When a name is later sold, `apps/portal`'s only resolver-related transfer safeguard — the "Detach the resolver" toggle, which defaults to checked — calls `registry.setResolver(tokenId, 0x0)`. That is a pointer flip on the **registry** contract; it never touches the **resolver** contract's role storage. The seller's root grant is therefore never revoked by the transfer flow — which contains no revoke step of any kind — and the buyer cannot remove it afterwards: `revokeRootRoles` reverts for any account that does not already hold the matching admin role, and the app's one page that *can* revoke is gated on holding that role, which only the seller does. (The grant is not unrevocable in the absolute sense — the seller can revoke it at will, and our PoC proves that as a negative control. What no in-scope path does is revoke it *as part of a transfer*, and what the buyer cannot do is force it.) A name still points at the seller's resolver right after a sale — the precondition for the seller's surviving grant to actively redirect records — in at least three proven populations: names whose holder lacks `ROLE_SET_RESOLVER`, so the toggle never renders; every name registered or migrated through `apps/manager`, which has no transfer UI and so never runs the detach step at all; and any Explorer transfer where the user unchecks the pre-checked toggle. In every other Explorer-driven transfer the toggle fires by default and does flip the pointer away — but because that pointer flip never touches the resolver's role storage, the seller's authority merely goes dormant rather than disappearing, and a later re-point of the same resolver (an additional, hypothetical condition nothing in-scope facilitates) revives it instantly with no new grant required. We state the two cases separately rather than collapsing them: in populations 1 and 2 — names whose holder lacks `ROLE_SET_RESOLVER`, so the toggle is hidden, and names registered or migrated through `apps/manager`, which has no transfer UI at all — the ex-owner can redirect the name's `addr(60)` (or any other record) immediately, with no on-chain signal that this authority exists and no action the buyer can take to remove it. In the default Explorer path the same authority survives but lies dormant behind the cleared registry pointer, and becomes exploitable again only if the name is ever pointed back at that resolver. The severity argument below rests on the first two populations, not on the dormant case.

### Vulnerability Details

**Root cause 1 — the registration entry point grants root, not name-scoped, roles.** `PermissionedResolver.authorizeNameRoles(bytes toName, uint256 roleBitmap, address account, bool grant)` takes a `toName` argument specifically so a grant can be scoped to one name's resource. The in-scope registration builder never uses that scoping — it always passes the root name:

```ts
// packages/smart-account/src/providers/rhinestone/registration-calls.ts:321-330
// 6. authorizeNameRoles(hex"00", ROLES.ALL, wallet, true) -- every session registration
calls.push({
  to: params.resolver,
  value: 0n,
  data: encodeFunctionData({
    abi: permissionedResolverAuthorizeNameRolesSnippet,
    functionName: 'authorizeNameRoles',
    args: ['0x00', ROLES_ALL, params.wallet, true],
  }),
})
```

`ROLES_ALL` (`packages/smart-account/src/providers/rhinestone/manifest.ts:171-172`) is every nibble-packed role bit set, i.e. every permission (`ROLE_SET_ADDR`, `ROLE_SET_ALIAS`, and every other resolver role plus their admin bits — see `apps/portal/src/lib/roles/resolverRoles.ts:24-46` for the bit layout). `'0x00'` is the resolver's **root** resource, not `dnsEncode(name)` for the name being registered.

**Root cause 2 — migration repeats the exact same choice.** `apps/manager`'s migration flow builds the identical call for every wallet it adds as a co-admin:

```ts
// apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:422-433
const buildWalletCoAdminCall = (params: {
  readonly resolver: Address
  readonly wallet: Address
}): Call => ({
  to: params.resolver,
  data: encodeFunctionData({
    abi: permissionedResolverAuthorizeNameRolesSnippet,
    functionName: 'authorizeNameRoles',
    args: [ROOT_NAME, ROLES_ALL, params.wallet, true],   // ROOT_NAME = '0x00' :44
  }),
  value: 0n,
})
```

**Scope framing.** Resolver-global root roles are the `PermissionedResolver` contract's own designed access model — `authorizeNameRoles` supports a root-resource grant by design, and a root grant being writable against any name the resolver serves is how the contract is built to work. That design choice is not what this report claims is a bug, and the ENS smart contracts are out of scope regardless. What is in scope, and what this report is about, is the **app's** choices: `packages/smart-account` and `apps/manager` choose to call `authorizeNameRoles` with the root resource (`'0x00'` / `ROOT_NAME`) instead of the name-scoped resource the very same function accepts, and `apps/portal`'s transfer flow then presents a remedy that — by its own file header docstring (`detachNameResolver.ts:1-14`, quoted below) — provably never removes that authority. Relatedly, "the buyer cannot self-remediate" below is a claim specifically about `revokeRootRoles` reverting for a fresh account, not a claim that no account could ever revoke the grant — the seller, or any existing holder of the matching admin role, still can.

**This is resolver-global, not per-name — proven live.** Root roles are read with `hasRootRoles(uint256 roleBitmap, address account)`, which has no name/node parameter at all. On the fork, `hasRootRoles(1 /* ROLE_SET_ADDR */, holder)` and `hasRootRoles(1<<28 /* ROLE_SET_ALIAS */, holder)` both read `true` for the real per-HCA resolver `0x197d52be85bdaba259f35f09bc074b2562cb1610`. Calling `setAddr` on `namehash("totally-unrelated-xyz123.eth")` — a node the holder never registered through any app flow — **succeeds**, and the node now resolves to the holder's chosen address. Membership in "names this wallet registered" is irrelevant; the only thing that matters is whether a name's registry entry currently points at this resolver contract.

**`apps/portal`'s transfer safeguard is a registry pointer flip, not a resolver-role revocation.** The Explorer's "Detach the resolver" option is described to the user as making the recipient's name resolve cleanly, with no mention that any prior authority over the resolver itself survives:

```ts
// apps/portal/src/features/transfer/components/SendNameForm.tsx:46-49 (OPTIONS entry)
key: 'detachResolver',
label: 'Detach the resolver',
description:
  'Detaches this name’s resolver so it stops resolving to your records entirely. The recipient starts clean and sets up their own.',
```

and its actual on-chain effect, per its own file header comment (`apps/portal/src/features/transfer/helpers/detachNameResolver.ts:1-14`):

> Detach a v2 name from its resolver (`registry.setResolver(tokenId, 0x0)`). … It never touches the resolver contract itself, which may be shared by the sender's other names — it only clears the registry's resolver pointer.

The comment is accurate about what the code does, but the UI copy never tells the user that "clearing the registry's resolver pointer" leaves the seller's grant on that resolver **fully intact** — it is not a name-scoped grant that dies with the pointer, it is a standing root permission on a contract the seller (not the buyer) controls. `buildTransferPlan.ts` confirms `detach-resolver` is the only resolver-related step this app ever builds — the full step set is `set-eth-addr | detach-resolver | detach-registry | transfer-token` (`buildTransferPlan.ts:18-42`) — so there is no `authorizeNameRoles(..., false)` and no `revokeRootRoles(...)` anywhere in the transfer plan.

**What the codebase-wide grep does and does not prove — stated precisely, because the obvious version of this claim is wrong.** Grepping all five in-scope trees for `revokeRootRoles` returns zero matches. That proves only that the *string* is absent from in-scope source. It does **not** prove that no in-scope code path ever emits the call, because the calldata is assembled inside `@ensdomains/ensjs` (pinned at `e96662c` via the `pnpm-workspace.yaml` catalog), not in this repo:

```ts
// @ensdomains/ensjs e96662c — src/actions/wallet/v2/revokeResolverRoles.ts:112-131
switch (params.scope) {
  case 'root':
    return { ...base, functionName: 'revokeRootRoles',
             args: [encodeResolverRoleBitmap(params.roles), params.targetAccount] }
  case 'name':
    return { ...base, functionName: 'authorizeNameRoles',
             args: [dnsEncode(params.name), encodeResolverRoleBitmap(params.roles),
                    params.targetAccount, false] }
```

`apps/portal`'s resolver Roles table therefore **does** build both calls at runtime: its "Remove user" action (`ResolverRolesSidebar.tsx:307-323` → `revokeResolverRoles.ts:45-60`) emits `revokeRootRoles` whenever the selected row resolves to the root scope, which is every root-only row — i.e. **all 112 of the 112 role rows that exist on live Sepolia today** (census below). So a revoke capability exists in-scope. The narrower claims this report actually rests on, each verified separately, are:

* **(a) The transfer flow never revokes.** No revoke step exists in `buildTransferPlan.ts`, and `detachNameResolver.ts`'s own docstring says the one step that runs "never touches the resolver contract itself."
* **(b) `apps/manager` has no transfer feature at all**, so names registered or migrated there are sold outside both in-scope apps and no detach step runs either.
* **(c) The page that can revoke is gated on the role the buyer will never hold.** `canManageRoles = Boolean(hasRootRole)` where `hasRootRole` is `hasRootRoles(['ROLE_SET_ADDR'], connectedAccount)` (`routes/resolver/$address/roles/index.tsx:40-50`). After the sale that predicate is true for the **seller** and false for the **buyer**. The roles UI renders read-only for the new owner: the party with the motive to revoke cannot, and the party who can has no prompt, no notification, and no reason to.

**The buyer cannot self-remediate on-chain either.** Calling `revokeRootRoles(1, holder)` from a fresh account with no roles reverts with `EACCannotRevokeRoles(0, 1, ...)` — root roles can only be revoked by an existing holder of the corresponding admin role, which the buyer is not. As a **negative control we ran ourselves** rather than leave for triage to find: the same call *from the holder* succeeds, `hasRootRoles(1, holder)` flips `true → false`, and our PoC asserts on that too (Section 4 of the transcript). We say so explicitly because "permanent and unrevocable" would be an overstatement, and the finding does not need it.

### Precondition, stated honestly

This is not a bug that fires on every transfer unconditionally — it requires that the transferred name **still point at the seller's per-HCA resolver after the transfer**, because that is what lets the seller's surviving grant actively resolve records for that specific name going forward. The claim being made is that this precondition holds by default for specific, proven populations of names (points 1–3 below), and that even in the Explorer-driven case where the one toggle meant to prevent it defaults to on and fires (point 4), it does not revoke the underlying authority — it only defers the authority's exercise until the same resolver is pointed at again. Concretely:

1. **The detach toggle is hidden entirely when the owner lacks `ROLE_SET_RESOLVER`.** Its visibility gate:

   ```ts
   // apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts:86-90
   detachResolver:
     resolverQuery.isSuccess &&
     hasResolver &&
     rolesQuery.isSuccess &&
     heldRoles.includes('ROLE_SET_RESOLVER'),
   ```

   The sibling helper's own docstring explains why a real, normally-transferred name can fail this gate — "This is a registry write, so the registry gates it on the owner holding `ROLE_SET_RESOLVER` for the token; a normally-registered owner auto-holds it, but a migrated/locked name can own the token yet lack it" (`apps/portal/src/features/transfer/helpers/detachNameResolver.ts:9-11`; the hook file quoted above only documents the parallel `ROLE_SET_SUBREGISTRY` case, for the sibling `detachRegistry` option, not this one). For the `ROLE_SET_RESOLVER`-lacking class of name the option never renders — the user is given no way to detach the resolver even if they wanted to, and no indication that anything was skipped.
2. **Where it is shown, it defaults to checked** (`SendNameForm.tsx:71`, the initial `useState` for `detachResolver` is `true`) — it fires by default and does flip the registry pointer; unchecking it is the only way this specific toggle causes the pointer to persist. But the warning shown when it is off never mentions the persistent issue — that the resolver's root role grant survives regardless of the toggle's state — it only says the name "could still be listed as the primary name for an address that no longer controls it."
3. **`apps/manager` has no transfer feature at all.** `find apps/manager/src -iname '*transfer*'` returns only a post-transfer notification component (`apps/manager/src/features/notifications/notifications/name-transferred.tsx`) — no transfer flow. Anyone transferring a Manager-registered name must do so outside both in-scope apps: directly on Etherscan, through a marketplace, or via the official ENS app. None of those paths execute this repo's `detachNameResolver` — the resolver pointer is untouched by default.
4. **Even when the toggle is shown, defaulted on, and exercised, the underlying grant is never revoked** (Root cause / PoC Section 3 below) — so the precondition is not even a permanent shield for the one path that tries: the same resolver, still holding the seller's `ROLES_ALL`, remains deployed and re-attachable. **Additional enabling condition, hypothetical, not facilitated in-scope:** if a name is later pointed back at that resolver — the buyer re-attaching it, or a completely different name being pointed at it by anyone via an ordinary `setResolver` call — full seller authority over that name revives instantly, with no new grant required. Nothing in-scope suggests or automates this re-pointing; it is not a natural continuation of the transfer flow, and the finding does not depend on it for populations 1–3 above, where the pointer is never flipped away in the first place.

### Impact Details

**In-scope entry point:** `packages/smart-account`'s registration call builder (`registration-calls.ts:321-330`) and `apps/manager`'s migration co-admin grant (`buildAtomicMigrationBatches.ts:422-433`) are where the app chooses to grant global root roles instead of a name-scoped grant; `apps/portal`'s transfer flow (`SendNameForm.tsx` / `useTransferDetachTargets.ts` / `detachNameResolver.ts`) is where the app presents "Detach the resolver" as the safeguard against exactly this, while provably leaving the authority in place.

Selected impacts. **Primary: "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient"** — specifically its own first sub-bullet, *"resolving or rendering an ENS name to an attacker-controlled address."* A name's `addr(60)` record is the address third parties send funds to when paying "alice.eth"; `ROLE_SET_ADDR` (bit 0 of `ROLES_ALL`) is exactly the permission that writes it, and a surviving root grant lets the ex-owner set it at will. **Secondary: "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer)"** — its sub-bullet *"a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker"* is root causes 1 and 2 exactly: the registration/migration is the step that hands out resolver-root `ROLES_ALL`. To be precise about the seam rather than paper over it: the *grant* is made inside a Manager flow, while the *harm* lands after a transfer that (per the precondition section) often happens outside both in-scope apps. The primary impact carries the report on its own; the secondary is named because the grant's origin is squarely a Manager flow. A seller who has sold "alice.eth" — believing (per the app's own UI copy) that they no longer control it, or never even seeing an option to relinquish anything — retains the ability to call `setAddr` on that name for as long as it points at their resolver, silently redirecting anyone who pays the buyer at that name.

**Severity we are claiming: High.** We are deliberately not claiming Critical. The unconditional populations (toggle hidden; Manager-registered names transferred outside the Explorer) carry the finding on their own, while the default Explorer path leaves the authority dormant rather than immediately exploitable — and we would rather anchor a defensible High on what is proven than have the whole report discounted for overclaiming on the conditional case.

**Blast radius, measured against live data rather than asserted.** We enumerated the v2 resolvers the staging indexer returns, de-duplicated them to unique resolver addresses, and read every role grant on each (`resolvers(first:100)` → `resolver(id:<address>)`; transcript in `evidence/role-scope-census.txt`). Result, re-run on 2026-09-06: **112 role grants across 60 live per-HCA resolvers, 112 of them root-scoped — not one name-scoped grant exists.** Every resolver the app has deployed has handed its wallet global authority over every name that resolver will ever serve. This is the design in `registration-calls.ts:321-330` observable in production, not a theoretical branch. (The exact counts drift by a few rows between runs as the indexer's first-100 window moves; the 100%-root proportion has been stable across every run we have taken.)

Breakdown of production impact:

* **No expiry, and not revocable by the party who needs it revoked.** Compare with R2-03 (session-key co-owner): that is an *accepted design residual* with a *stated 7-day lifetime*, and the carve-out explicitly excludes duplicates that merely restate the broad scope — "authority beyond its stated lifetime… would be a new finding." This finding is exactly that: there is no lifetime here at all. The grant never expires, nothing in the transfer flow revokes it, the one in-scope UI presented to the seller as the remedy ("Detach the resolver") does not revoke it, and the buyer is locked out of the only page that could — both by the contract (`EACCannotRevokeRoles`) and by the app's own `canManageRoles` gate. The seller *can* revoke it voluntarily; nothing ever asks them to, and the buyer cannot make them.
* **Silent — no signal to the buyer.** Nothing in the buyer's UI communicates that the name's resolver carries a standing root grant to the previous owner. The buyer has no reason to check, and no in-scope tool to check with.
* **Full record control, not just `addr`.** `ROLES_ALL` includes every resolver permission (text records, content hash, ABI, pubkey, alias, plus every admin bit) — `addr(60)` is the most directly fund-relevant one, but the seller's authority is total.
* **Applies to every HCA registration and every migration**, not an edge case: the salt that determines the resolver's address is a pure function of the HCA address alone (`manifest.ts:223-236`, `computeResolverSalt`), and `resolverDeployed` (checked at `registration-calls.ts:248`) means the *same* resolver is reused for every subsequent registration by that HCA — this is the standard path, not a misconfiguration.
* **No privileges, no special timing.** The seller does not need to race the transfer, monitor mempools, or hold any special role beyond the one every HCA registration and migration already grants it as a matter of course.

### Why this is not a duplicate

* **QA-01** ("Certain role combinations when transferring a name… cause errors when later updating profile records or setting a primary name") is about **functional errors**, and its carve-out is explicit: "**A demonstrated security consequence (e.g., a role combination that grants unintended authority) would be a new finding.**" This report is precisely that demonstrated security consequence — a role combination (root-scoped `ROLES_ALL`, surviving `setResolver(0x0)`) that grants unintended, continuing, buyer-unrevocable authority to the seller after transfer. Concretely, the combination is: the seller's surviving resolver-root `ROLES_ALL` grant coexisting with the buyer's new registry ownership of the same name — two authorities over the same name held by two different accounts at once, which is exactly the shape QA-01's own carve-out names. To be precise about the difference rather than paper over it: QA-01's own trigger is transferring a name (or a name with subnames) and its own symptom is functional errors on later profile-record or primary-name updates; this report's trigger is the same transfer step but its symptom is a security one — unauthorized write authority, not an error — so while both sit under "role combinations surfaced by a transfer," they are not the same observed behavior.
* **WEB-1257** ("After a parent name is transferred, its registry is not detached, and the previous owner can no longer set a subname they still hold as their primary name") is about the **registry/subregistry** side of a *parent* name and a **functional** consequence (a *legitimate* primary-name-setting action failing) — the opposite direction from this report, which is about the **resolver**'s root role grant surviving on *any* name (parent or leaf) and giving the *previous* owner continued **write** authority, a security consequence, not a usability one.
* **WEB-1249** ("creating a new subname can replace the existing child registry… orphaning nested subnames") is a subregistry-replacement defect scoped to subname creation; it shares no mechanism with a resolver-level root-role grant surviving a name transfer.
* **R2-03** (session-key co-owner, accepted design residual) is a *different authority object* — an ephemeral ERC-4337 session-validator key, not a resolver role — with an explicit, accepted, **stated 7-day lifetime**, and the program's own carve-out for it is that authority *beyond that lifetime* is a new finding. This resolver root grant has no stated lifetime at all — it never expires and no transfer path removes it. It is also not the "co-owner" the R2-03 write-up describes; it is a resolver-contract role, orthogonal to smart account ownership.
* **R2-06** (Low) — "Session 'revocation' only clears browser storage; no on-chain revocation is performed, so the co-owner permission persists until it expires, making the 7-day lifetime the real bound." A triager could try to fold this report into R2-06 as another instance of the same shape — a "revoke"/"detach" step presented to the user that does not actually revoke. But R2-06's own text prices that residual as *bounded*: the 7-day expiry is what R2-06 itself calls "the real bound." The resolver root grant this report describes has no expiry of any kind — nothing in-scope ever times it out — so it falls outside R2-06's own priced residual even taking R2-06 entirely on its own terms.
* **F-2** (sibling report, `report-2-package`) is the resolver's *other* independently fixable defect and is not a duplicate of this one. F-2 is that a revoke path **does** exist — the Portal's resolver Roles table has a "Remove user" action — but it builds its revoke calldata against the wrong EAC resource: for an account holding both a root grant and a name grant, the helper always picks the name resource (`resolvedNames.find((n) => n !== '(root)') ?? ''`), never root, so "Remove user" reports success while the account's root authority survives untouched. F-3 (this report) is that the grant is made at registration/migration time and **the transfer flow never attempts to revoke it at all** — it has no "Remove user"-style step, only the registry-pointer-flipping "Detach the resolver" — while the one page that could revoke it is gated on a role only the seller holds. The two findings are complementary rather than overlapping: fixing F-2's resource argument makes "Remove user" correct for a seller who chooses to visit that page; it does nothing for a buyer who never can. Same subsystem (the `PermissionedResolver`'s root-scoped role storage), different defects (F-2: a revoke attempt with the wrong resource argument; F-3: no revoke attempt exists in the flow that matters here), different fixes (F-2: correct the resource argument passed to an existing call; F-3: add a revoke step, or scope the grant to a name, where neither exists today) — we file both because each is independently fixable and neither report's fix subsumes the other's.
* **SEC-TXM-002** ("A configured or cached smart-account address is returned without confirming it matches the address the SDK derives live") is about **address derivation/caching correctness** in the transaction manager — a different component, a different mechanism, and a different consequence (wrong address used as owner/target due to a stale cache) from a resolver's own role-storage surviving a transfer.

### References

* Root-cause 1 (registration grants global root, not name-scoped): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/smart-account/src/providers/rhinestone/registration-calls.ts>
* Root-cause 2 (migration repeats the same choice): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts>
* Salt/reuse (one resolver per HCA, forever): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/smart-account/src/providers/rhinestone/manifest.ts>
* Transfer safeguard that does not revoke (pointer flip only): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal/src/features/transfer/helpers/detachNameResolver.ts>
* Detach visibility gate (hidden without `ROLE_SET_RESOLVER`): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts>
* Detach toggle UI copy: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal/src/features/transfer/components/SendNameForm.tsx>
* Transfer plan (only resolver step is detach, never revoke): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal/src/features/transfer/utils/buildTransferPlan.ts>
* Resolver role bit layout: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal/src/lib/roles/resolverRoles.ts>

## Link to Proof of Concept

<https://gist.github.com/yoptascript/89f2b2ae2b6fb0b22f3b4783163cdc05>

## Proof of Concept

## Proof of Concept

Script: **`poc-F3.sh`** — <https://gist.github.com/yoptascript/89f2b2ae2b6fb0b22f3b4783163cdc05>. It starts its own anvil fork of live Sepolia, runs 22 assertions, and exits non-zero if any fails. Requires only `foundry` (`anvil` + `cast`) and a checkout of this repo at the audited commit. All addresses and transactions are the app's real Sepolia deployments and real, already-established on-chain history; every write runs only against the local fork, so nothing touches live state.

```bash
git clone https://github.com/immunefi-team/audit-comp-ens
git -C audit-comp-ens checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
REPO=./audit-comp-ens bash poc-F3.sh
# => exit 0, 22/22 OK, 0 FAIL

# Optional overrides. FORK_URL must serve anvil's fork bootstrap; the archive endpoint
# must still serve the resolver's deploy-tx receipt at block 11638766. Public Sepolia
# endpoints rate-limit and rotate their retained history, so the script tries several in
# order (each twice) and falls back to the fork RPC — if one run reports "did not serve
# the receipt", re-run or point ARCHIVE_RPC at your own archive node.
FORK_URL=https://ethereum-sepolia-rpc.publicnode.com \
ARCHIVE_RPC=https://1rpc.io/sepolia \
REPO=./audit-comp-ens bash poc-F3.sh
```

Note on the assertions: assertion 20 is a **negative control against our own claim** — it shows the holder's `revokeRootRoles` succeeding, so the report does not have to be taken on faith that the grant is "unrevocable." It is not. The finding is that no transfer path revokes it and the buyer cannot.

Full captured transcript (`evidence/F3-poc-fork-run.txt`):

F-3 PoC transcript -- global ROLES\_ALL grant survives every in-scope remedy Generated: 2026-09-07T18:53:32Z Repo pin: 1c9b47f18fcddd2e864dfe385c4171061c9811ae

\=== 0. ENVIRONMENT === Starting local anvil fork of live Sepolia on port 8630 (own port, not the shared 8555 fork). anvil pid=89581 rpc=<http://127.0.0.1:8630> chain-id=11155111 OK: chain-id is Sepolia (11155111) -- \[11155111] fork head block=11656030

\=== 1. THE GRANT EXISTS ON A REAL APP-CREATED INSTANCE === Resolver: 0x197d52be85bdaba259f35f09bc074b2562cb1610 Holder (HCA): 0x909BAd90d6Ec72cB1Ff9e37C8309B1C0b245f735 Deploy tx: 0x4fcc47918767369e9e44301b5a36769e825b70fde51e9b4ca1e91293ec8a8108 (block 11638766)

\-- EIP-1967 implementation slot on the resolver proxy -- impl slot raw: 0x0000000000000000000000009eae5c2730a7dd16bdd1dee6421a1b91e3b0365e OK: EIP-1967 impl == ensPermissionedResolverImpl -- \[0x9eae5c2730a7dd16bdd1dee6421a1b91e3b0365e]

\-- deploy tx receipt: VerifiableFactory ProxyDeployed log -- (receipt served by <https://1rpc.io/sepolia>, attempt 1) OK: receipt status success -- found \[status 1 (success)] OK: receipt shows a log emitted BY VerifiableFactory -- found \[0x10dc6333cdfe1fcef624c6e0a8221b91804cd7ef] OK: receipt shows the resolver-impl address in the deploy log data -- found \[9eae5c2730a7dd16bdd1dee6421a1b91e3b0365e]

\-- hasRootRoles(ROLE\_SET\_ADDR=1<<0, holder) -- hasRootRoles(1, holder) = true OK: holder holds ROLE\_SET\_ADDR at the RESOLVER ROOT (hex"00") -- true

\-- hasRootRoles(ROLE\_SET\_ALIAS=1<<28, holder) -- hasRootRoles(268435456, holder) = true OK: holder holds ROLE\_SET\_ALIAS at the RESOLVER ROOT -- true

Code responsible (in-scope app, not the ENS contract): packages/smart-account/src/providers/rhinestone/registration-calls.ts:321-330 // 6. authorizeNameRoles(hex"00", ROLES.ALL, wallet, true) -- every session registration calls.push({ to: params.resolver, value: 0n, data: encodeFunctionData({ abi: permissionedResolverAuthorizeNameRolesSnippet, functionName: 'authorizeNameRoles', args: \['0x00', ROLES\_ALL, params.wallet, true], }), })

apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:422-432 const buildWalletCoAdminCall = (params: {...}): Call => ({ to: params.resolver, data: encodeFunctionData({ abi: permissionedResolverAuthorizeNameRolesSnippet, functionName: 'authorizeNameRoles', args: \[ROOT\_NAME, ROLES\_ALL, params.wallet, true], // ROOT\_NAME = '0x00' (line 44) }), }) -- runs again for every co-admin wallet added during migration.

\=== 2. THE AUTHORITY IS A WILDCARD (root, not name-scoped) === namehash("totally-unrelated-xyz123.eth") = 0x6019558da875e65a6f3f0d56f2a17487fc84df3722d25bb3e63318bd4b6bde7a (the holder never registered this name through any app flow) OK: setAddr on a NEVER-registered node succeeds (no revert) -- found \[status 1 (success)] addr(unrelated\_node, 60) now reads: 0xdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef OK: the node the holder never owned now resolves to the holder's chosen address -- \[0xdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef]

Root roles are RESOLVER-GLOBAL (granted at hex"00"), not per-name. Any node whose registry entry ever points at this resolver inherits this authority -- membership in "names the holder registered" is irrelevant.

\=== 3. THE APP'S OWN "REMEDY" DOES NOTHING === Explorer transfer flow files read for this step: apps/portal/src/features/transfer/utils/buildTransferPlan.ts -- the only resolver-related transfer step is 'detach-resolver' (registry.setResolver(id, 0x0)); there is no re-authorize / revoke step. apps/portal/src/features/transfer/helpers/detachNameResolver.ts:1-13 /\*\* \* Detach a v2 name from its resolver (`registry.setResolver(tokenId, 0x0)`). \* ... \* It never touches the resolver contract itself, which may be shared by \* the sender's other names -- it only clears the registry's resolver \* pointer. \*/

We reproduce the exact primitive it calls (registry.setResolver, selector 0xbc7b6d62) against a name we register on the fork that uses this SAME resolver, to show the round trip end-to-end.

\-- registering "f3poc1788807227.eth" for a test owner, resolver = the SAME per-HCA resolver -- (impersonating the ETHRegistrar as msg.sender to call registry.register directly -- test setup only, on our own fork; grants the owner ROLE\_SET\_RESOLVER=1<<24 so it can exercise the Explorer's own detach step) OK: registration tx status=0x1 tokenId (from register event, topics\[2]) = 0x10b32402af88069025b303de6346b416c9a79f496e7426ac33db86d2f743b037

\-- holder writes a record on this brand-new node (via the shared resolver) -- OK: holder's write lands before any transfer step -- \[0x1111111111111111111111111111111111111111] hasRootRoles(1, holder) BEFORE detach = true

\-- Explorer step: registry.setResolver(id, 0x0) \["Detach the resolver"] -- OK: detach (setResolver -> 0x0) succeeds -- found \[status 1 (success)] hasRootRoles(1, holder) DURING detach = true OK: root role UNCHANGED while the name is "detached" -- true

\-- holder writes AGAIN, while the registry-level pointer is detached -- OK: holder's write succeeds even while "detached" -- found \[status 1 (success)] OK: the record is updated to the holder's new value -- \[0x2222222222222222222222222222222222222222]

\-- registry.setResolver(id, resolver) \[re-attach -- ADDITIONAL, HYPOTHETICAL condition; nothing in-scope facilitates it, e.g. buyer restores it] -- OK: re-attach succeeds -- found \[status 1 (success)] hasRootRoles(1, holder) AFTER re-attach = true OK: root role UNCHANGED after the full detach/reattach round trip -- true OK: the name resolves to the holder's redirect again once reattached -- \[0x2222222222222222222222222222222222222222]

VERDICT: registry.setResolver(id,0x0)+back never reads or writes any storage slot on the resolver contract -- it is a pointer flip in a DIFFERENT contract. hasRootRoles is identical at every step, and the holder's write authority is never interrupted for even one block.

\=== 4. THE BUYER CANNOT SELF-REMEDIATE === -- a fresh account with no roles calls revokeRootRoles(1, holder) -- Error: Failed to estimate gas: server returned an error response: error code 3: execution reverted: custom error 0xa604e318: 0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000feed9911, data: "0xa604e3180000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000feed9911": EACCannotRevokeRoles(0, 1, 0x00000000000000000000000000000000FEeD9911) OK: revokeRootRoles reverts with EACCannotRevokeRoles -- found \[EACCannotRevokeRoles]

\-- NEGATIVE CONTROL: the grant IS revocable, but only by an existing holder -- (the seller). We prove this ourselves rather than let it be found for us: status 1 (success) transactionHash 0xd99cb14b94ac7f7449ffe4989d7637f1188e24324f1d398835f9a16a6c501980 hasRootRoles(1, holder) after the HOLDER's own revokeRootRoles = false OK: the holder CAN revoke its own root grant (so it is not unrevocable in general) -- \[false] => the accurate claim is NOT 'nothing can revoke this'. It is: no TRANSFER-FLOW path revokes it, and the party who needs it revoked -- the buyer -- cannot. OK: fork state restored after the control (grant back in place) -- true

\-- grep: no in-scope SOURCE FILE contains the string revokeRootRoles -- grep exit status: 1 (1 = no matches found in any in-scope tree) matches: \[] OK: zero in-scope source-level occurrences of the string revokeRootRoles

IMPORTANT -- what that grep does and does NOT prove. It proves only that the STRING is absent from in-scope source. It does NOT prove no in-scope code path ever emits the call, because the calldata is built inside ensjs, not in this repo: @ensdomains/ensjs (pinned e96662c, pnpm-workspace.yaml catalog) src/actions/wallet/v2/revokeResolverRoles.ts:112-131 scope:'root' -> functionName 'revokeRootRoles' scope:'name' -> functionName 'authorizeNameRoles', args\[3] = false So apps/portal's resolver Roles table (ResolverRolesSidebar.tsx, 'Remove user') DOES build both calls at runtime, and for a root-only row it emits precisely revokeRootRoles. The correct, narrower claim -- the one this report makes -- is: (a) the TRANSFER FLOW never revokes: buildTransferPlan.ts's only resolver step is 'detach-resolver' (set-eth-addr | detach-resolver | detach-registry | transfer-token), and detachNameResolver.ts's own docstring says it 'never touches the resolver contract itself'; (b) apps/manager has no transfer feature at all (find apps/manager/src -iname '*transfer*' returns only a notification component), so its names are sold outside both in-scope apps and the detach step never runs; (c) the roles page that CAN revoke is gated on canManageRoles = hasRootRoles(\['ROLE\_SET\_ADDR'], connectedAccount) (routes/resolver/$address/roles/index.tsx:40-50), which the BUYER never satisfies -- so the only account that can clean this up is the seller, voluntarily, after the sale, with nothing prompting them to.

\=== 5. BLAST RADIUS === Precise structural bound (code-grounded, not a live subgraph query -- the indexer is out of scope and would only tell us the past, not the future):

packages/smart-account/src/providers/rhinestone/manifest.ts:223-236 computeResolverSalt(hca) = keccak256(abi.encode( keccak256("OwnedResolver"), hca, 0 /\* version, hard-coded \*/)) packages/smart-account/src/providers/rhinestone/registration-calls.ts:248 if (!params.resolverDeployed) { ...deployProxy(impl, salt, initData)... }

The salt is a pure function of the HCA address alone (fixed version=0). VerifiableFactory's CREATE2 address is therefore fully determined by (HCA, impl, factory) -- ONE resolver address per HCA, forever. Line 246's resolverDeployed guard means every subsequent registration BY THE SAME HCA reuses this exact resolver rather than deploying a new one. So the set of names sharing this resolver's root-role grant is exactly: every name this HCA has ever registered or migrated through the Manager, PLUS every name it ever will in the future -- a set that only grows and, per Sections 3-4, never shrinks.

That undercounts the true exposure, though: Section 2 already proved the authority is not scoped to "names this HCA registered" at all -- it is GLOBAL on the resolver contract. Any node whatsoever whose registry entry is pointed at 0x197d52be85bdaba259f35f09bc074b2562cb1610 (by anyone, via the ordinary setResolver path used above) inherits the holder's full ROLES\_ALL authority. The bound on "how many names could be affected" is therefore not a property of the resolver at all -- it is unbounded and set entirely by registry configuration, which the app itself writes on every registration/migration/transfer.

Concrete redirect already demonstrated in Section 2: addr(namehash("totally-unrelated-xyz123.eth"), 60) = 0xdeadbeef... (holder-chosen) addr(namehash("f3poc1788807227.eth"), 60) = 0x2222... (holder-chosen, survives detach/reattach)

\=== 6. VERDICT === VERDICT: CONFIRMED -- every HCA registration/migration grants the wallet global root ROLES\_ALL on its PermissionedResolver (registration-calls.ts:321-330, buildAtomicMigrationBatches.ts:422-432); the Explorer's only transfer-time remedy (setResolver(id,0x0), detachNameResolver.ts) never touches that grant (hasRootRoles identical before/during/after, holder writes throughout); the transfer flow contains no revoke step at all; and a buyer's own revokeRootRoles call reverts with EACCannotRevokeRoles while the SELLER's succeeds -- so after a sale the seller retains write authority over every name that points at this resolver, all the way down to redirecting addr(60), and the buyer has no way to remove it. The grant has no expiry and no transfer-flow revocation; it is not unrevocable in the absolute sense.

**Expected:** either (a) registration/migration scope the grant to the specific name (`authorizeNameRoles(dnsEncode(name), ...)` instead of root `'0x00'`), so a role is only ever exercisable on names actually owned by the grantee, or (b) the transfer flow actually revokes the grant it is trying to protect against, so "Detach the resolver" does what its own description promises.

**Actual:** the grant is made at the resolver's root resource, is resolver-global rather than name-scoped, is never revoked by any step in the transfer flow (and is not revocable at all by the buyer, who fails the `canManageRoles` gate on the one page that could), and the one UI control presented as the safeguard against it (`apps/portal`'s "Detach the resolver") only flips a registry pointer that the resolver's role storage does not even read.

**Suggested fix:**

1. **In `registration-calls.ts` and `buildAtomicMigrationBatches.ts`, replace the root resource `'0x00'`/`ROOT_NAME` with `dnsEncode(name)` (or the equivalent per-name resource) in the `authorizeNameRoles` calls** so the wallet's write authority is scoped to the name actually being registered/migrated, not the whole resolver. **This fix, alone, is incomplete — it does not remove the root grant.** The deploy step already grants root unconditionally, and does so at a different call that `authorizeNameRoles` has nothing to do with: `packages/smart-account`'s builder calls `PermissionedResolver.initialize(params.hca, ROLES_ALL, [])` (`registration-calls.ts`, the `deployProxy`/`initialize` call at :255-266), and that `initialize` function has **no name/node parameter at all** — it is structurally incapable of a name-scoped grant, so no amount of editing `authorizeNameRoles`'s argument touches it. The migration flow has the identical twin call, `buildResolverDeploymentCall` (`buildAtomicMigrationBatches.ts:396-420`), which builds the same root-scoped `initialize(params.hca, ROLES_ALL, [])` for every migrated HCA. There are therefore **two grantees to track, not one**: the HCA itself, granted root at `initialize` time, and the wallet EOA granted root by the separate `authorizeNameRoles` call this fix item targets — and after a sale, **both** remain under the seller's control regardless of what happens to the second grant, which is why the finding stands even if item 1 ships in isolation. (The migration code's own verification step treats even its "wallet-name-roles" check with `hasRootRoles` rather than a name-scoped `hasRoles` read — `verifyAtomicMigrationBatch.ts:227-244`, where the `'wallet-name-roles'` case at :238-244 calls the identical `hasRootRoles` function used by the `'resolver-root-roles'` case one switch-arm above it, and both gate on `expectation.rootName === '0x00'` — evidence that the authors' own tooling treats these as root grants too, not name-scoped ones.) A complete fix needs either (i) a per-name resolver deployed per registration rather than one shared resolver per HCA, (ii) a revocation step for the `initialize`-time HCA grant wired into the transfer plan, or (iii) a contract-side change to `initialize` to accept a name scope. Swapping the argument in `authorizeNameRoles` alone leaves the `initialize`-time HCA grant fully intact.
2. Independently, before or during the Portal's transfer flow, emit an `authorizeNameRoles(nameOrRoot, ROLES_ALL, seller, false)` (or `revokeRootRoles`) call covering **both** the wallet EOA and the HCA so "Detach the resolver" actually revokes the authority its own UI copy claims to remove, rather than only clearing the registry's resolver pointer.
3. Until (1)/(2) ship, treat any name whose resolver is (or was ever) a per-HCA `PermissionedResolver` as carrying standing seller authority (held by both the HCA and its wallet EOA) that transfer alone does not remove, and surface that to the user explicitly instead of the current "starts clean" language.
