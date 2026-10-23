# 91848 w a critical editing a record scoped resolver delegate silently submits grantrootroles authority over every name on the resolver

## #91848 \[W\&A-Critical] Editing a record-scoped resolver delegate silently submits \`grantRootRoles\` — authority over every name on the resolver

**Submitted on Sep 4th 2026 at 17:59:30 UTC by @Naresh\_Kandula for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91848
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

### Description

## Editing a record-scoped resolver delegate silently submits `grantRootRoles` — authority over every name on the resolver

**Target:** Explorer app (`apps/portal`) — <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal> (fork of `ensdomains/apps-monorepo` @ `63772fd872af472ced58b009499355f3430c2a86`) **Severity: Critical** **Impact in scope (one clause):** *"Malicious interactions with an already-connected wallet, such as: **modifying transaction arguments or parameters** … inducing a blanket token approval (for example `setApprovalForAll` on the registry or NameWrapper) **to an attacker-controlled address**."*

***

### On severity, stated up front rather than left for triage

This is not a display finding, and it is deliberately **not** filed under the "display or pre-fill an incorrect address, name, or transaction detail" clause. The wrong value is never rendered anywhere. `roleName` has exactly two consumers and **both are transaction builders** (`ResolverRolesSidebar.tsx:299` and `:318`); it is the `name` argument of an on-chain write. The application hands the connected wallet a transaction whose encoded function is **`grantRootRoles`** — every name on the resolver — while the operator is managing a delegate scoped to a single record on a single name. That is *modifying transaction arguments or parameters*, natively. There is no execution seam to argue about: the defect **is** the submitted transaction.

The clause's own example is the same shape — a **blanket** authority grant where a **scoped** one was intended, landing on a third party's address. `ROLE_SET_ADDR` held at root is a direct fund-redirection primitive: the delegate can repoint the ETH address record of **any** name using that resolver.

**The one precondition, priced here rather than hidden.** The transaction is signed by a resolver admin performing routine delegate management: open the delegate's row → adjust roles → **Save changes**. Three things about that:

1. **It is the normal app workflow.** Editing a delegate's roles is the entire purpose of `/resolver/$address/roles`. The out-of-scope exclusion for *"un-prompted, in-app user actions that are not part of the normal app workflows"* therefore does not apply.
2. **The attacker induces it with an ordinary request** — "could you also give me contenthash?" Nothing unusual is asked of the admin.
3. **The admin is shown no scope indication at all while doing it.** The one UI element that would reveal scope is suppressed in exactly the failing case — `ResolverRolesSidebar.tsx:391`, guarded on `resolvedNames.length > 0`.

**The trigger is the safe configuration.** The attacker needs no exotic setup. They need only to hold the **narrowest** role the system can express. Using least privilege is what arms the escalation.

***

### 1. One-paragraph summary

ENS v2's `PermissionedResolver` scopes a role to `resource(node, part)`, where `part` namespaces the record type — so a delegate can hold `ROLE_SET_TEXT` over *just the `avatar` text record of one name*. That is the least-privilege configuration the contract is designed to express, and `@ensdomains/ensjs` ships the helpers to compute it. The portal only ever computes the **`part = 0`** resource, so a record-scoped grant is never in its resource→name map. `resolvedNames` comes back empty, and the code path that handles "no name" is the *same one* that means **"root — all names."** An admin who opens that delegate's row and clicks **Save changes** issues `grantRootRoles(...)`, handing the delegate their selected roles across every name on the resolver. The same defect makes **Remove user** issue `revokeRootRoles(...)`, which never touches the delegate's actual record-scoped grant — so the delegate disappears from the admin's view and keeps their access.

### 2. Root cause — one line

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169`

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`?? ''` conflates **"this resource is not in our map"** with **"the operator chose root scope."** Downstream, `''` is a *documented sentinel* — `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:20`:

```ts
/** Dotted name (e.g. "myname.eth") or empty string for ROOT_RESOURCE (all names). */
readonly name: string
```

### 3. The chain — every link verified in production source

| # | Site                                                                               | Behaviour                                                                                                                                                                                                       |
| - | ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | `lib/roles/resolverRoles.ts:138-141`                                               | `computeNameResource(name)` = `keccak256(encodePacked(['bytes32','bytes32'],[namehash(name), ZERO_BYTES32]))` — **`part` hardcoded to 0**. Own comment: *"Mirrors `PermissionedResolverLib.resource(node, 0)`"* |
| 2 | `lib/roles/resolverRoles.ts:147-160`                                               | `buildResourceToNameMap` maps only `ROOT_RESOURCE → '(root)'` plus `computeNameResource(node.name)` per node. A `part ≠ 0` resource is **never in the map**                                                     |
| 3 | `lib/roles/resolverRoles.ts:179-200`                                               | `groupRolesByAccount` looks up `role.resource.toLowerCase()`; a miss ⇒ `resolvedNames === []`                                                                                                                   |
| 4 | `ResolverRolesSidebar.tsx:391-397`                                                 | The scope hint is guarded by `resolvedNames.length > 0` ⇒ **nothing renders**. The admin gets no indication of scope                                                                                            |
| 5 | `ResolverRolesSidebar.tsx:169`                                                     | `roleName = ''`                                                                                                                                                                                                 |
| 6 | `ResolverRolesSidebar.tsx:299` (`handleSaveChanges`) · `:318` (`handleRemoveUser`) | both pass `name: roleName`                                                                                                                                                                                      |
| 7 | `helpers/grantResolverRoles.ts:46-59` · `helpers/revokeResolverRoles.ts:47-60`     | identical selector: `name === '' ? { scope:'root' } : { scope:'name', name }`                                                                                                                                   |
| 8 | ensjs `grantResolverRolesWriteParameters` / `revokeResolverRolesWriteParameters`   | `scope:'root'` ⇒ **`grantRootRoles` / `revokeRootRoles`** (all names). `scope:'name'` ⇒ `authorizeNameRoles` (one name)                                                                                         |

### 4. Consumer list — this is not a display defect

`roleName` has exactly **two** consumers, and **both are transaction builders**: `handleSaveChanges` (`ResolverRolesSidebar.tsx:299`) and `handleRemoveUser` (`:318`). It is **never rendered**. The value whose derivation is wrong is the `name` argument of an on-chain write, so the impact clause is satisfied natively.

### 5. Downstream guards — enumerated, none stop it

The only validation anywhere on this path is `if (roles.length === 0) throw` (`grantResolverRoles.ts:40`, `revokeResolverRoles.ts:42`), which concerns roles, not names. There is **no** check that an empty name was *intended*, because an empty name is a legitimate documented input meaning root. `prepareGrantResolverRolesTransaction` validates only `walletClient.account` and `walletClient.chain`. Nothing re-derives the scope from the resource, and nothing compares the encoded `functionName` against what the UI displayed. The transaction reaches the transaction manager already encoded.

### 6. Reachability — why a `part ≠ 0` role exists and reaches the client

Three measurements, because this is the premise the finding stands on:

1. **It is a first-class shipped primitive.** `@ensdomains/ensjs` @ `e96662c` ships `dist/utils/v2/roles/resolverResource.js`, re-exported from the public entry point `@ensdomains/ensjs/public/v2`, exporting `computeResolverResource(node, part)`, `textPart(key)` and `addrPart(coinType)` — each documented in its own JSDoc as mirroring `PermissionedResolverLib.resource` / `partHash` in Solidity. Per-record scoping is the contract's intended least-privilege mechanism, not a hypothetical. **The PoC imports these real helpers.**
2. **The indexer returns those rows.** `features/resolver/hooks/useResolverOverview.ts:90-98` queries `roles { account resource roleBitmap … }` with **no `where` filter on `resource`**, so `part ≠ 0` rows reach the client like any other.
3. **The row is rendered and editable.** `ResolverRolesTable.tsx:68-83` renders the Name cell as `null` when `resolvedNames.length === 0` — the row still appears — and `showActions = canManageRoles && !disableEdit` (`:107`) attaches the edit action to it.

***

### Proof of Concept

### 7. Proof of concept

Self-contained. **No network, no wallet, no RPC.** Added to the repo at:

```
apps/portal/src/lib/roles/poc-record-scope-root-escalation.test.ts
```

**Nothing is hand-rolled.** Every hash comes from `@ensdomains/ensjs` (`computeResolverResource`, `textPart`, `addrPart`) and `viem/ens` (`namehash`); the bigint→hex conversion is the app's **own** `toResourceHex`; the grouping and mapping are the app's **own** production functions (`buildResourceToNameMap`, `computeNameResource`, `groupRolesByAccount`, `resolverRoles`); and the encoded call is produced by ensjs's **real** write-parameter builders — the same ones `grantResolverRoles.ts` and `revokeResolverRoles.ts` call. The only mirrored lines are the two derivations that live inline inside a React component and cannot be imported; both are reproduced verbatim with their `file:line` above them.

#### Steps to reproduce

```bash
# 1. clone the competition fork at the audited commit
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout audit-comp-ready

# 2. install
pnpm install

# 3. save the PoC (Appendix A) to:
#    apps/portal/src/lib/roles/poc-record-scope-root-escalation.test.ts

# 4. run it
pnpm --filter portal exec vitest run \
  src/lib/roles/poc-record-scope-root-escalation.test.ts --reporter=verbose
```

#### Actual output

```
 RUN  v4.1.5 /…/apps/portal

stdout | src/lib/roles/poc-record-scope-root-escalation.test.ts > EXPLOIT … > EVIDENCE …

DELEGATE 0x4444444444444444444444444444444444444444 on resolver 0x2222222222222222222222222222222222222222
admin opens the row and clicks "Save changes" / "Remove user"

  delegate is scoped to | scope shown to admin      | encoded call        | remove encodes
  ----------------------+---------------------------+---------------------+----------------
  whole name (part = 0) | Roles scoped to alice.eth | authorizeNameRoles  | authorizeNameRoles
  text 'avatar'         | (nothing rendered)        | grantRootRoles      | revokeRootRoles
  addr coinType 60      | (nothing rendered)        | grantRootRoles      | revokeRootRoles

 ✓ CONTROL — whole-name (part = 0) delegate > resolves to the name, shows the scope hint, and encodes NAME-scoped 4ms
 ✓ CONTROL — whole-name (part = 0) delegate > the app's own resource arithmetic is byte-identical to ensjs's 1ms
 ✓ EXPLOIT — record-scoped delegate collapses to ROOT > text record 'avatar': unresolvable -> no hint -> grantRootRoles (ALL NAMES) 1ms
 ✓ EXPLOIT — record-scoped delegate collapses to ROOT > address record coinType 60 (ETH): unresolvable -> no hint -> grantRootRoles (ALL NAMES) 0ms
 ✓ EXPLOIT — record-scoped delegate collapses to ROOT > ORACLE: the escalation is a real widening — root !== the name resource 0ms
 ✓ EXPLOIT — record-scoped delegate collapses to ROOT > EVIDENCE — what the admin sees vs what the wallet is asked to sign 2ms

 Test Files  1 passed (1)
      Tests  6 passed (6)
```

#### The single most important line in this report

Read the middle column against the fourth. For the two record-scoped rows the admin is shown **nothing at all** about scope, and the wallet is asked to sign **`grantRootRoles`** — every name on the resolver. The first row is the control: when the portal *can* name the scope, it says so and encodes `authorizeNameRoles`, one name.

*(On the control's fourth column: a name-scoped revoke is expressed as `authorizeNameRoles` carrying the reduced bitmap — the contract's own idiom — whereas the root path has a dedicated `revokeRootRoles`. The control row is correct; the divergence is confined to the two record-scoped rows.)*

#### What each test proves

| Test                                                  | Proves                                                                                                                                                                                         |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CONTROL · resolves and encodes name-scoped            | With a nameable scope the app behaves correctly: hint rendered, `authorizeNameRoles`. Establishes the PoC is not vacuous                                                                       |
| CONTROL · resource arithmetic byte-identical to ensjs | The portal's `computeNameResource` equals ensjs's `computeResolverResource(node, 0)` via the app's own `toResourceHex`. Makes every exploit row's resource trustworthy — no hand-rolled crypto |
| EXPLOIT · `textPart('avatar')`                        | `resolvedNames === []` → no hint → `roleName === ''` → **`grantRootRoles`**, and Remove encodes **`revokeRootRoles`**                                                                          |
| EXPLOIT · `addrPart(60n)`                             | Same, for an address record — the `ROLE_SET_ADDR` fund-redirection case                                                                                                                        |
| ORACLE · root ≠ name resource                         | The collapse is a real **widening**, not a relabelling: the record-scoped resource is a different value from the name resource                                                                 |
| EVIDENCE                                              | Prints the shown-vs-signed table above, from the real functions                                                                                                                                |

***

### 8. Impact

* **Privilege escalation to every name on the resolver.** A delegate intentionally limited to one record on one name receives their selected roles — `ROLE_SET_ADDR`, `ROLE_SET_TEXT`, `ROLE_SET_CONTENTHASH`, … — across **all** names using that resolver. `ROLE_SET_ADDR` at root is a direct fund-redirection primitive: the delegate can repoint the ETH address record of any of those names to an address they control.
* **The safest configuration is the vulnerable one.** The attacker's only requirement is to hold the *narrowest* role the system can express. Adopting least privilege is what arms the escalation.
* **No warning is possible for the admin.** The scope hint is suppressed exactly when `resolvedNames` is empty — the one element that would reveal scope is absent precisely in the failing case. The admin sees a delegate row with a blank Name cell and an edit button.
* **Revocation does not revoke.** "Remove user" emits `revokeRootRoles`, which does not clear the delegate's record-scoped grant. The row vanishes from the admin's view while the delegate retains their original authority — so the defect also destroys the admin's ability to detect or undo it.
* **Cost to the attacker: nothing.** They accept a narrow role and make one ordinary follow-up request.

### 9. Why this is not a known issue

Checked against all 60 disclosed records (25 security findings, 10 QA items, 25 WEB tracker entries).

**QA-01** is the nearest, and its published carve-out is precisely this finding:

> *"a role combination that **grants unintended authority** rather than merely erroring would be a new finding."*

* **Same root cause?** No. QA-01 concerns role *combinations* producing downstream *errors* in the transfer flow ("cause various errors when later updating profile records or setting a primary name"). This is a resource-resolution failure in the resolver-roles sidebar collapsing write scope onto root.
* **Materially different invariant?** Yes: *a role write must be addressed to the scope the operator was shown.*
* **New security consequence?** Yes. QA-01 errors; this grants authority to a third party and then silently fails to revoke it.

**WEB-1249** (creating a subname can detach an existing child registry) is a different flow, a different contract, and involves no role grant. **WEB-1257 / QA-01** (cannot set primary name after parent transfer) is a registry detachment, not a scope collapse. **R2-03 / R2-06** concern the smart-account session key, not resolver roles. Distinct from previously filed **#89363** (migration re-minting a role for a stale v1 manager — different flow, different root cause, different app).

On the 19 bare `WEB-*` identifiers that publish no behaviour: the six that are described are product defects (a commit loop, missing gas warnings, a stale wrapper expiry, a detached child registry) — the character of a UX tracker. This report does not claim a malfunction. It claims the application encodes a **wider authority grant than the operator was shown**, to a third party, and that the corresponding revoke does not undo it. If ENS believes a tracker entry covers it, that entry will name `ResolverRolesSidebar.tsx` or the `resolvedNames`→`roleName` derivation; nothing in the published set does.

### 10. Recommendation

Distinguish **"unresolvable"** from **"root."** They are currently the same value.

Carry the resource through to the write instead of round-tripping through a display name — keep `resource` on the row and select `scope: 'root'` only when `resource === ROOT_RESOURCE`:

```ts
// ResolverRolesSidebar.tsx — instead of `resolvedNames.find(...) ?? ''`
const scope = row.original.resource === ROOT_RESOURCE
  ? ({ kind: 'root' } as const)
  : resolvedNames.length > 0
    ? ({ kind: 'name', name: resolvedNames.find((n) => n !== '(root)')! } as const)
    : ({ kind: 'unresolvable' } as const)   // <-- new third outcome
```

Failing that, **refuse to build a transaction when `resolvedNames.length === 0`**, and render the raw resource with an explicit "scope could not be determined" warning — the inverse of the current behaviour, which renders nothing and proceeds. Longer term, adopt ensjs's `computeResolverResource(node, part)` / `textPart` / `addrPart` so record-scoped grants are nameable in the UI at all.

### 11. Verification checklist for triage

Shortest path to confirming this without building anything:

* [ ] `ResolverRolesSidebar.tsx:169` — confirm `?? ''`.
* [ ] `grantResolverRoles.ts:20` — confirm the doc comment declares `''` to mean ROOT\_RESOURCE / all names.
* [ ] `grantResolverRoles.ts:46` and `revokeResolverRoles.ts:47` — confirm the identical `name === ''` selector.
* [ ] `ResolverRolesSidebar.tsx:299` and `:318` — confirm both pass `name: roleName`, and that `roleName` is never rendered.
* [ ] `ResolverRolesSidebar.tsx:391` — confirm the scope hint is guarded on `resolvedNames.length > 0`.
* [ ] `lib/roles/resolverRoles.ts:138-141` — confirm `part` is hardcoded to `ZERO_BYTES32`.
* [ ] `useResolverOverview.ts:90-98` — confirm the roles query has no `where` filter on `resource`.
* [ ] Run the PoC: `pnpm --filter portal exec vitest run src/lib/roles/poc-record-scope-root-escalation.test.ts --reporter=verbose` — 6/6, and read the printed shown-vs-signed table.

***

### Appendix A — full PoC source

`apps/portal/src/lib/roles/poc-record-scope-root-escalation.test.ts`

```ts
/**
 * PoC — editing a RECORD-SCOPED resolver delegate silently submits
 *       grantRootRoles / revokeRootRoles: authority over EVERY name
 *
 * NOTHING IS HAND-ROLLED IN THIS FILE.
 *   - every hash comes from @ensdomains/ensjs (`computeResolverResource`,
 *     `textPart`, `addrPart`) and viem (`namehash`);
 *   - the bigint->hex conversion is the app's OWN `toResourceHex`;
 *   - grouping/mapping are the app's OWN production functions;
 *   - the encoded call is produced by ensjs's real write-parameter builders.
 * The only mirrored lines are the two derivations that live inline inside a
 * React component and cannot be imported; both are reproduced verbatim with
 * file:line above them.
 *
 * No network. No wallet. No RPC.
 */
import {
  addrPart,
  computeResolverResource,
  textPart,
} from '@ensdomains/ensjs/public/v2'
import {
  grantResolverRolesWriteParameters,
  revokeResolverRolesWriteParameters,
} from '@ensdomains/ensjs/wallet/v2'
import { namehash } from 'viem/ens'
import { describe, expect, it } from 'vitest'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from './resolverRoles'
import { toResourceHex } from './toResourceHex'

/** The resolver being administered. */
const RESOLVER = '0x2222222222222222222222222222222222222222'
/** The delegate whose row the admin opens — the party that gains authority. */
const DELEGATE = '0x4444444444444444444444444444444444444444'
/** One of the names on this resolver. */
const NAME = 'alice.eth'

const ZERO32 =
  '0x0000000000000000000000000000000000000000000000000000000000000000' as const

// The connected admin's wallet. Only `chain` + `account` are read by the
// ensjs write-parameter builders; no transport, no RPC.
// biome-ignore lint/suspicious/noExplicitAny: minimal viem wallet-client double
const adminWallet: any = {
  chain: { id: 11155111 },
  account: { address: '0x1111111111111111111111111111111111111111' },
}

// --- the two production derivations that live inline in the component -------

/** ResolverRolesSidebar.tsx:169 — verbatim */
const roleNameOf = (resolvedNames: readonly string[]) =>
  resolvedNames.find((n) => n !== '(root)') ?? ''

/** ResolverRolesSidebar.tsx:391-397 — verbatim, incl. the `length > 0` guard */
const scopeHintOf = (resolvedNames: readonly string[]) =>
  resolvedNames.length === 0
    ? '(nothing rendered)'
    : resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`

/**
 * grantResolverRoles.ts:46-59 and revokeResolverRoles.ts:47-60 — the identical
 * `name === ''` selector, driving ensjs's real encoders.
 */
const encodeSave = (name: string, roles: string[]) =>
  grantResolverRolesWriteParameters(
    adminWallet,
    (name === ''
      ? { resolverAddress: RESOLVER, targetAccount: DELEGATE, scope: 'root', roles }
      : {
          resolverAddress: RESOLVER,
          targetAccount: DELEGATE,
          scope: 'name',
          name,
          roles,
        }) as Parameters<typeof grantResolverRolesWriteParameters>[1],
  )

const encodeRemove = (name: string, roles: string[]) =>
  revokeResolverRolesWriteParameters(
    adminWallet,
    (name === ''
      ? { resolverAddress: RESOLVER, targetAccount: DELEGATE, scope: 'root', roles }
      : {
          resolverAddress: RESOLVER,
          targetAccount: DELEGATE,
          scope: 'name',
          name,
          roles,
        }) as Parameters<typeof revokeResolverRolesWriteParameters>[1],
  )

/**
 * Runs the REAL production grouping over a single indexer row, exactly as
 * `useResolverOverview` delivers it (`roles { account resource roleBitmap }`,
 * no `where` filter on `resource`).
 */
const groupFor = (resourceHex: string) =>
  groupRolesByAccount(
    [
      {
        account: DELEGATE,
        resource: resourceHex,
        roleBitmap: resolverRoles.ROLE_SET_TEXT.toString(),
      },
    ],
    buildResourceToNameMap([{ name: NAME }]),
  )[0]

/** ensjs resource -> the hex form the indexer/grouping layer uses. */
const resourceHex = (part: `0x${string}`) =>
  toResourceHex(computeResolverResource(namehash(NAME), part))

// ===========================================================================
// CONTROL — a whole-name (part = 0) delegate behaves correctly
// ===========================================================================

describe('CONTROL — whole-name (part = 0) delegate', () => {
  it('resolves to the name, shows the scope hint, and encodes NAME-scoped', () => {
    const g = groupFor(computeNameResource(NAME))

    expect(g.resolvedNames).toEqual([NAME])
    expect(scopeHintOf(g.resolvedNames)).toBe(`Roles scoped to ${NAME}`)
    expect(roleNameOf(g.resolvedNames)).toBe(NAME)
    expect(encodeSave(NAME, ['ROLE_SET_TEXT']).functionName).toBe(
      'authorizeNameRoles',
    )
  })

  it("the app's own resource arithmetic is byte-identical to ensjs's", () => {
    // Both sides are real implementations: the portal's `computeNameResource`
    // and ensjs's `computeResolverResource(node, 0)` via the app's own
    // `toResourceHex`. This is what makes the exploit rows trustworthy.
    expect(computeNameResource(NAME).toLowerCase()).toBe(
      resourceHex(ZERO32).toLowerCase(),
    )
  })
})

// ===========================================================================
// EXPLOIT — a record-scoped delegate collapses to ROOT
// ===========================================================================

describe('EXPLOIT — record-scoped delegate collapses to ROOT', () => {
  const CASES = [
    ["text record 'avatar'", textPart('avatar')],
    ['address record coinType 60 (ETH)', addrPart(60n)],
  ] as const

  for (const [label, part] of CASES) {
    it(`${label}: unresolvable -> no hint -> grantRootRoles (ALL NAMES)`, () => {
      const resource = resourceHex(part)

      // 1. the portal cannot name this resource — it only ever maps part = 0
      const g = groupFor(resource)
      expect(g.resolvedNames).toEqual([])

      // 2. the admin is shown NOTHING about scope (the hint is length-guarded)
      expect(scopeHintOf(g.resolvedNames)).toBe('(nothing rendered)')

      // 3. the sidebar derives the empty name (`?? ''`)
      const roleName = roleNameOf(g.resolvedNames)
      expect(roleName).toBe('')

      // 4. "Save changes" encodes a ROOT grant over every name
      const save = encodeSave(roleName, ['ROLE_SET_TEXT'])
      expect(save.functionName).toBe('grantRootRoles')
      expect(save.functionName).not.toBe('authorizeNameRoles')
      expect(save.args[0]).toBe(resolverRoles.ROLE_SET_TEXT)

      // 5. "Remove user" encodes a ROOT revoke — which never clears the
      //    delegate's actual record-scoped grant. Displayed as removed,
      //    still authorised.
      expect(encodeRemove(roleName, ['ROLE_SET_TEXT']).functionName).toBe(
        'revokeRootRoles',
      )
    })
  }

  it('ORACLE: the escalation is a real widening — root !== the name resource', () => {
    expect(resourceHex(textPart('avatar')).toLowerCase()).not.toBe(
      computeNameResource(NAME).toLowerCase(),
    )
    expect(resourceHex(addrPart(60n)).toLowerCase()).not.toBe(
      computeNameResource(NAME).toLowerCase(),
    )
  })

  it('EVIDENCE — what the admin sees vs what the wallet is asked to sign', () => {
    const rows = [
      { scope: 'whole name (part = 0)', resource: computeNameResource(NAME) },
      { scope: "text 'avatar'", resource: resourceHex(textPart('avatar')) },
      { scope: 'addr coinType 60', resource: resourceHex(addrPart(60n)) },
    ].map(({ scope, resource }) => {
      const names = groupFor(resource).resolvedNames
      const roleName = roleNameOf(names)
      return {
        scope,
        shown: scopeHintOf(names),
        signed: encodeSave(roleName, ['ROLE_SET_TEXT']).functionName,
        removes: encodeRemove(roleName, ['ROLE_SET_TEXT']).functionName,
      }
    })

    const w = Math.max(...rows.map((r) => r.scope.length))
    const h = Math.max(...rows.map((r) => r.shown.length))
    console.log(
      `\nDELEGATE ${DELEGATE} on resolver ${RESOLVER}\n` +
        `admin opens the row and clicks "Save changes" / "Remove user"\n\n` +
        `  ${'delegate is scoped to'.padEnd(w)} | ${'scope shown to admin'.padEnd(h)} | encoded call        | remove encodes\n` +
        `  ${'-'.repeat(w)}-+-${'-'.repeat(h)}-+---------------------+----------------\n` +
        rows
          .map(
            (r) =>
              `  ${r.scope.padEnd(w)} | ${r.shown.padEnd(h)} | ${r.signed.padEnd(19)} | ${r.removes}`,
          )
          .join('\n') +
        '\n',
    )

    // the control names its scope; both record-scoped rows show nothing and
    // silently widen to root
    expect(rows[0].signed).toBe('authorizeNameRoles')
    expect(rows[1].signed).toBe('grantRootRoles')
    expect(rows[2].signed).toBe('grantRootRoles')
  })
})
```
