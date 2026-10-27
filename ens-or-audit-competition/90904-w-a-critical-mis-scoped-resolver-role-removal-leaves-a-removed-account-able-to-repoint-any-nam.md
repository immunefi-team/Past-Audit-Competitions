# 90904 w a critical mis scoped resolver role removal leaves a removed account able to repoint any name the resolver serves

**Submitted on Aug 28th 2026 at 01:27:05 UTC by @webrainsec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90904
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: resolving or rendering an ENS name to an attacker-controlled address; showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer's resolver role editor merges every scope an account holds into a single table row, then writes changes back to only one of those scopes. When an account holds roles at both the resolver root resource and a per-name resource, "Remove user" emits `authorizeNameRoles` against one name instead of `revokeRootRoles`. The transaction confirms and the interface reports the user was removed from all roles, while root `ROLE_SET_ADDR` stays live.

The attached fork PoC registers a real `.eth` name wired to the resolver in the ENS registry, then has the removed account repoint it: `wrsecpoc1.eth` resolves to the owner before and to an attacker-controlled address after, read back through `registry.getResolver()` then `resolver.addr()`.

Two bounds up front. The account must have been a legitimately authorized role holder, so this is the incident-response case. And a second "Remove user" does clear the root role, so the window lasts until the operator repeats the action or re-checks.

## Vulnerability Details

The PermissionedResolver grants roles per resource: a root resource covering every name, and a per-name resource. The Explorer's own "Add user" sheet offers both, defaulting to root (`ResolverAddUserSheet.tsx:39` defines `ROOT_NODE_VALUE = ''`, `:127` seeds the state with it, and the scope selector is at `:251-258` with a "Root (all nodes)" option alongside the per-node list). Holding roles at more than one resource is an ordinary supported configuration.

`groupRolesByAccount` collapses that distinction. It keys its map on `account` alone and unions everything else (`apps/portal/src/lib/roles/resolverRoles.ts:194-205`):

```ts
const account = role.account.toLowerCase()
const resource = role.resource.toLowerCase()
const resolvedName = resourceToName?.get(resource) ?? null
const existing = grouped.get(account)          // keyed on account ONLY
if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)       // union across ALL resources
  if (resolvedName) existing.resolvedNames.add(resolvedName)
}
```

The edit sidebar then discards every scope but one (`ResolverRolesSidebar.tsx:169`):

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`roleName` is the only scope that reaches the chain. `handleRemoveUser` submits it with the merged role union (`:307-321`), and the write helper branches on it (`revokeResolverRoles.ts:50-61`):

```ts
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

Those branches are different contract entrypoints, not two arguments to one. ensjs maps `scope: 'root'` to `revokeRootRoles` and `scope: 'name'` to `authorizeNameRoles` (`@ensdomains/ensjs/dist/actions/wallet/v2/revokeResolverRoles.js:25,32`). A name-scoped call cannot clear a root grant.

The interface states the correct scope and then does the other thing. `buildResourceToNameMap` maps the root resource to `'(root)'` (`resolverRoles.ts:151`), so for an account holding both scopes `resolvedNames` is `['(root)', 'a.eth']`. The header therefore renders "Global roles (all names)" (`:393-395`) while `roleName` resolves to `'a.eth'`, and the confirm dialog reads "Are you sure you want to remove this user from all roles? This action cannot be undone." (`:445-447`).

The Save path has the same defect: unchecking a permission runs `computeRoleChanges` over the merged `decodedRoles` (`:190-192`) and revokes it against `roleName`.

**Recovery.** After the first removal the row carries only the root resource, so `resolvedNames` is `['(root)']`, `roleName` falls back to `''`, and a second "Remove user" takes the root branch and clears the grant.

Two variants follow from the same line. With two name scopes and no root, only the first is written. With `resolvedNames` empty while `decodedRoles` is not, reachable for a part-scoped `textPart` or `addrPart` role since `computeNameResource` only computes `resource(node, 0)` (`:138-141`), `roleName` falls back to `''` and a grant the operator believes is name-scoped is written with `grantRootRoles`. That second variant is an escalation rather than a failed removal; I have not built it end to end.

Which scope gets picked is also not deterministic: `resolvedNames` is a `Set` in indexer-return order.

## Impact Details

The victim is the operator of a PermissionedResolver and the owners of every name it serves. Root `ROLE_SET_ADDR` is the authority to call `setAddr` for any node on that resolver, so while the operator believes the removal succeeded, the removed account can repoint any of those names and payments resolved through them go to the attacker. Exposure is every name the resolver serves, for the duration of the window. Stated in scope rather than dollars because the program is on Sepolia.

The window is bounded by operator attention, not by the contract. No explicit error, warning, or semantic verification identifies the partial removal. The surviving root row does remain visible after the refresh, so an attentive operator can notice and remove the user again; one who trusts the success message does not. The roles page is the only revocation surface in the Explorer.

The selected wallet-interaction impact covers this even though the app is not hostile: its own example, "when the request's target chain is not validated", is a first-party validation defect rather than attacker-injected calldata. Here the same shape applies, the operator's connected wallet signs `authorizeNameRoles(a.eth, ...)` while the interface states "Global roles (all names)".

Both bounds from the Brief apply: the actor is a formerly authorized role holder, and a second removal closes the window.

## References

* `apps/portal/src/lib/roles/resolverRoles.ts:179-221` (`groupRolesByAccount`), merge at `:194-205`
* `apps/portal/src/lib/roles/resolverRoles.ts:147-160` (`buildResourceToNameMap`), root mapping at `:151`
* `apps/portal/src/lib/roles/resolverRoles.ts:138-141` (`computeNameResource`)
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169` (the defect)
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:307-321` (`handleRemoveUser`), `:190-192` (Save path), `:393-395` (header), `:445-447` (dialog copy)
* `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:39` (`ROOT_NODE_VALUE`), `:127` (default scope), `:251-258` (scope selector)
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:50-61` (scope branch)
* `apps/portal/src/routes/resolver/$address/roles/index.tsx:41-50` (page, gated on root `ROLE_SET_ADDR`)
* `@ensdomains/ensjs/dist/actions/wallet/v2/revokeResolverRoles.js:25,32` (scope to function mapping)
* PermissionedResolver implementation `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`, VerifiableFactory `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef`, Sepolia

## Proof of Concept

Secret gist contains the two PoC scripts. All state changes happen on a local anvil fork of Sepolia pinned to block 11578783. Nothing is sent to a live network and no repository file is modified.

{% stepper %}
{% step %}

## `poc_resolver_roles_onchain.sh` (primary, self-contained)

Needs only foundry and a Sepolia RPC. It registers a real `.eth` name through the deployed ETHRegistrar wired to a freshly deployed PermissionedResolver, grants an account roles at both scopes, submits the exact transaction the Explorer submits for "Remove user", then has the removed account repoint the name and reads resolution back through the registry.

Requires a fresh fork, because the salt and label are fixed so the addresses below are reproducible. The script checks this and says so rather than reverting opaquely.

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --fork-block-number 11578783 --port 8547
bash poc_resolver_roles_onchain.sh
```

```
SETUP  prepare two plain EOAs on the fork
  operator  0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
  alice     0x1563915e194D8CfBA1943570603F7606A3115508

SETUP  register a real .eth name wired to that resolver
  resolver proxy  0xafb27d89102bf97080d741E21d8F43Da4F375fc5
  name            wrsecpoc1.eth
  registry.getResolver("wrsecpoc1") = 0xafb27d89102bf97080d741E21d8F43Da4F375fc5
  resolves to     0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A   (the owner)

STEP 1  grant Alice roles at TWO scopes, both offered by the Add user sheet
  ok    grantRootRoles(ROLE_SET_ADDR, alice)                   root scope, ALL names
  ok    authorizeNameRoles(a.eth, ROLE_SET_TEXT, alice, true)  name scope

STEP 2  the operator clicks "Remove user"
  -> authorizeNameRoles(a.eth, ROLE_SET_ADDR|ROLE_SET_TEXT, alice, false)
  ok    transaction confirmed. The interface reports removal from all roles.
  hasRootRoles(ROLE_SET_ADDR, alice) = true

STEP 3  IMPACT: Alice repoints the registered name
  wrsecpoc1.eth resolved to  0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
  wrsecpoc1.eth now resolves 0x000000000000000000000000000000000000dEaD

STEP 4  recovery
  -> revokeRootRoles(ROLE_SET_ADDR, alice)
  hasRootRoles(ROLE_SET_ADDR, alice) = false
  ok    Alice can no longer write records

RESULT: PASS
```

Step 3 reads resolution as a client does, `registry.getResolver(label)` then `resolver.addr(node)`. Step 4 is the bound.

Fork-setup note: every well-known test key, the anvil defaults included, carries an EIP-7702 delegation on Sepolia at this block, which fails the registry's ERC-1155 receiver check on `register`. The script clears it on its two accounts and asserts the clear succeeded. That is setup, not part of the vulnerability. The `anvil_*` calls use plain JSON-RPC rather than `cast rpc`, which aborts intermittently on those methods.
{% endstep %}

{% step %}

## `poc_resolver_roles_app.mjs` (supporting: why the wrong call is chosen)

Runs the Explorer's own `groupRolesByAccount` and `buildResourceToNameMap`, plus the real ensjs `revokeResolverRolesWriteParameters` the app calls, all resolved from the repository checkout passed on the command line. Requires the monorepo with `pnpm install` already run. No network access is needed: it uses an `esbuild` already present in the checkout to strip the module's types.

```bash
node poc_resolver_roles_app.mjs /path/to/apps-monorepo
```

```
1. groupRolesByAccount collapses two resources into one row
   rows          : 1
   decodedRoles  : [ 'ROLE_SET_ADDR', 'ROLE_SET_TEXT' ]   <- union across BOTH resources
   resolvedNames : [ '(root)', 'a.eth' ]

2. what the sidebar tells the operator (Sidebar.tsx:393-395)
   header        : "Global roles (all names)"

3. the scope it actually picks (Sidebar.tsx:169)
   roleName      : "a.eth"   <- root scope discarded

4. the call the app sends vs the call the root grant needs
   sent          : authorizeNameRoles
   required      : revokeRootRoles

5. after the first removal the row holds only the root resource,
   so roleName falls back to '' and a SECOND removal does clear it
```

{% endstep %}

{% step %}

## Through the interface

1. Open `/resolver/<resolver>/roles` as an account holding root `ROLE_SET_ADDR`.
2. Add user: Alice, scope "Root (all nodes)", check Set Address.
3. Add user: Alice, scope `a.eth`, check Set Text.
4. Reload. Alice is one row; the Name column reads `(root) a.eth`.
5. Open the edit sidebar. The header reads "Global roles (all names)".
6. Click Remove user, confirm, wait for the transaction, and observe the success state.
7. Read `hasRootRoles(ROLE_SET_ADDR, alice)` on the resolver. It is still true.
8. Click Remove user a second time. It now clears.
   {% endstep %}
   {% endstepper %}

## Recommended fix

The indexer already returns the per-role scope: `ResolverRole` carries both `resource` and `name` (`features/resolver/hooks/useResolverOverview.ts:14-22`, queried at `:90-98`). None of it is used; the sidebar reconstructs the scope from a node-name map instead.

{% stepper %}
{% step %}

## Preserve resource scope per row

Key `groupRolesByAccount` on `(account, resource)` so one row is one resource, or keep the merged row for display and pass the specific `ResolverRole` with its `resource` into the sidebar.
{% endstep %}

{% step %}

## Drive role writes from the resource

Drive `grantResolverRoles` and `revokeResolverRoles` from `role.resource` directly rather than re-deriving a scope from a name.
{% endstep %}

{% step %}

## Use a discriminated scope union

Make the scope a discriminated union, `{ scope: 'root' } | { scope: 'name'; name }`, instead of inferring root from `name === ''`. The current empty-string default is fail-open: an unresolvable scope silently becomes the most privileged one.
{% endstep %}

{% step %}

## Revoke every spanned resource

Until those land, make "Remove user" emit one transaction per resource the row spans, and do not report success until all of them confirm.
{% endstep %}
{% endstepper %}
