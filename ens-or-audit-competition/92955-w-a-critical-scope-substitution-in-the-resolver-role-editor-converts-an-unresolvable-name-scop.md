# 92955 w a critical scope substitution in the resolver role editor converts an unresolvable name scoped role into a resolver root grant giving unintended global authority over every name on the r&#x20;

**Submitted on Sep 13th 2026 at 13:30:08 UTC by @Kodekloner43926 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92955
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user funds
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The ENS Explorer's resolver role editor derives the scope of a write transaction from a display-name lookup instead of the stored resource identifier. When a role's resource hash can no longer be matched to a name, the lookup yields nothing, the editor silently represents that as the empty string, and the transaction builder interprets the empty string as resolver root. An administrator who opens such a row and adds a permission therefore signs a `grantRootRoles` transaction: a role that exists for one name becomes authority over every current and future name using the resolver. With `ROLE_SET_ADDR` this enables replacement of address records for unrelated names, redirecting payments to an attacker-controlled address.

## Vulnerability Details

Affected asset: Explorer app (`apps/portal`), frozen revision `63772fd872af472ced58b009499355f3430c2a86`. The vulnerable flow is the resolver user-management page (`apps/portal/src/routes/resolver/$address/roles/`) used by administrators of a Permissioned Resolver.

**1. An unresolvable resource still produces a fully editable row.** Every role entry carries a resource hash identifying its scope. `groupRolesByAccount` in `apps/portal/src/lib/roles/resolverRoles.ts` (lines 179–221) attempts to recover a human-readable name for each resource and merges entries per account:

```ts
for (const role of roles) {
  const account = role.account.toLowerCase()
  const resource = role.resource.toLowerCase()
  const decoded = decodeResolverRoleBitmap(role.roleBitmap)
  const resolvedName = resourceToName?.get(resource) ?? null

  const existing = grouped.get(account)

  if (existing) {
    existing.roles.push(role)
    existing.decodedRoles.push(...decoded)
    if (resolvedName) existing.resolvedNames.add(resolvedName)
  } else {
    const names = new Set<string>()
    if (resolvedName) names.add(resolvedName)
    grouped.set(account, {
      account,
      resolvedNames: names,
      roles: [role],
      decodedRoles: [...decoded],
    })
  }
}
```

When the lookup misses, the account and its decoded permissions survive intact, but the resource that defined the scope is dropped from the row's model — `resolvedNames` ends up empty. Nothing marks the row as degraded; it remains listed and editable like any other.

**2. The editor maps "no name found" to the root sentinel.** `ResolverRolesSidebar.tsx` derives the write scope from the display names (line 169):

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The save handler then submits that derived value with the edit (lines 294–304):

```ts
const handleSaveChanges = () => {
  if (!selectedAccount || saveMutation.isPending) return
  setPendingAction({
    type: 'save',
    name: roleName,
    account: selectedAccount,
    rolesToGrant: rolesToGrant,
    rolesToRevoke: rolesToRevoke,
  })
  openModal()
}
```

**3. The transaction builder gives that value privileged meaning.** `grantResolverRoles.ts` (lines 46–62) treats the empty string as an explicit root selection:

```ts
const writeParams = grantResolverRolesWriteParameters(
  walletClient as Parameters<typeof grantResolverRolesWriteParameters>[0],
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
)
```

The empty string is a deliberate sentinel in this application: the **Add user** flow defines `ROOT_NODE_VALUE = ''` for users who explicitly choose root scope. The bug is that the same value also arrives from a failed lookup. Two materially different states — "the administrator chose Root (all nodes)" and "the application could not determine this role's scope" — collapse into one branch at the transaction boundary, and the ambiguous case fails open to the most privileged scope. The final wallet request is `grantRootRoles(bitmap, account)` on the resolver, even though the stored role's resource is a non-root, per-name resource.

**How the unresolvable state arises (no malformed chain state required):**

* **Resolver migration:** a name such as `moved-away.eth` is scoped to a shared permissioned resolver; a contractor holds a role on that name's resource; the name later switches to a different resolver (a normal ENS operation the Explorer itself supports). The old resolver's role record persists, but its node list no longer contains the name — the resource-to-name map built from that list cannot resolve the row.
* **Record-type-scoped grants (unresolvable from creation):** the resolver's public interface supports `authorizeTextRoles`/`authorizeAddrRoles`, whose resources are computed as `resource(node, part)` with a nonzero part (the keccak256 of the text key or coin type). The Explorer's map is built exclusively from part-zero resources — `computeNameResource` mirrors `resource(node, 0)` — so any such grant renders as an unresolvable row for its entire lifetime, without any migration ever occurring.

Either path places a legitimate, still-indexed, still-editable role row in front of the administrator with no indication that its scope has been lost.

**Root cause in one sentence:** the editable model discards the resource identifier of unresolved non-root roles, and the write path uses a single untagged string both for explicit root selection and for lookup failure, so the transaction layer cannot distinguish the two and defaults to root.

**Recommended fix.** Carry the resource hash on every editable row and model scope explicitly:

```ts
type ResolverRoleScope =
  | { type: 'root' }
  | { type: 'name'; name: string }
  | { type: 'unresolved'; resource: Hex }
```

Only `{ type: 'root' }` may produce `grantRootRoles`. A row whose non-root resource cannot be mapped should display its resource hash, be read-only for name-based edits, and route to an explicit resource-aware recovery flow. Reject any absent or unresolved scope before a transaction is prepared, and require a visible **Root (all names)** confirmation before any root grant is signed. Add a regression test that supplies a nonzero resource absent from the map and asserts no grant transaction can be prepared for it.

## Impact Details

Selected impacts: **Critical — malicious interactions with an already-connected wallet (modifying transaction arguments or parameters)** (primary), and **Critical — direct theft of user funds** (secondary). The scope substitution in the transaction the connected wallet signs is the primary impact; the resulting unintended root authority opens the theft path through address-record redirection.

**Attack path:**

1. An organization runs several production names through one permissioned resolver.
2. A contractor account holds `ROLE_SET_TEXT` scoped to `moved-away.eth` only.
3. `moved-away.eth` migrates to another resolver; the contractor's role entry remains on the original resolver and its row becomes unresolvable.
4. An administrator later reviews the resolver's users. The contractor's row is still listed — the role record exists — but shows no resolved name.
5. The administrator opens the row and enables **Set Address**, intending to adjust that limited account, and saves.
6. The application submits the edit with the empty-name sentinel; the connected wallet receives `grantRootRoles(bitmap, contractor)` addressed to the resolver.
7. After the administrator signs, the contractor holds global `ROLE_SET_ADDR` and replaces the ETH address of an unrelated production name with an attacker-controlled address.
8. Payments resolved through that name are sent to the attacker.

**Risk breakdown:**

* **Victim:** an authorized resolver administrator using the normal role-management interface.
* **Attacker prerequisite:** the target account holds any name-scoped or record-type-scoped role whose resource is absent from the resolver's current resource-to-name map. No attacker interaction is needed during the escalation itself.
* **Required victim interaction:** the administrator adds at least one permission on that visible row and signs the generated transaction.
* **Complexity:** low once the row exists; the sentinel conversion is deterministic.
* **Privileges gained:** the newly selected permissions are granted at resolver root instead of the original name scope.
* **Security boundary crossed:** authority expands from one name resource to every current and future name on the resolver.
* **Maximum consequence:** address-record replacement for unrelated names and redirection of payments sent to them.

**Why this is Critical rather than a lower severity:**

* **The escalation requires no attacker action.** It occurs during an ordinary permission edit by an administrator; the target account only needs the pre-existing role row.
* **The wallet display is not informed consent.** The wallet may show the function name `grantRootRoles`, but the application's own confirmation layer — the row the administrator opened and the modal labeled "Update roles for \<account>" — never indicated that a name-scoped row would be submitted as a global grant. Holding administrators responsible for reconciling ABI function names against the UI's scope presentation defeats the purpose of the scope-display layer the application itself maintains.
* **Post-hoc visibility does not reverse the grant.** The root grant lands on-chain and is immediately exercisable; noticing the row now renders as global does not revoke anything.

**Relation to published known issues.** The published QA-01 entry concerns functional errors when certain role combinations are edited and expressly excludes demonstrated unintended authority from its scope. This report demonstrates exactly such a privilege-boundary failure: a lookup miss for a non-root resource converted into resolver-root authority. No other published R2, R3, SEC, EXP, WEB, or QA entry describes this transition. It is also distinct from a companion finding about merged-row removal revoking only one resource: that path drops scope boundaries while revoking an account whose resources are all known; this path fails open from one unknown non-root resource into a new global grant during an edit. Different trigger, different direction (grant versus revoke), independently remediable.

## References

* Affected files (frozen revision `63772fd872af472ced58b009499355f3430c2a86`): `apps/portal/src/lib/roles/resolverRoles.ts` (lines 179–221), `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx` (line 169; lines 294–304), `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts` (lines 46–62)

## Proof of Concept

## Proof of Concept

### Part A — Step-by-step (application flow)

1. A contractor account holds `ROLE_SET_TEXT` for `moved-away.eth` on a shared permissioned resolver (granted through the resolver roles page at that name's scope, or directly on-chain).
2. `moved-away.eth` changes to a different resolver — a normal ENS operation the Explorer supports. The old resolver's role entry persists while its node list drops the name, so the contractor's row becomes unresolvable. (Equivalently, a record-type-scoped grant made through the resolver's public interface — `authorizeTextRoles`/`authorizeAddrRoles` — creates a nonzero-part resource that the application's part-zero name map can never resolve; such a row is unresolvable from the moment it is created, with no migration involved.)
3. An administrator opens the old resolver's roles page (`/<resolver-address>/roles`). The contractor's row is listed but shows no resolved name, and remains editable.
4. The administrator opens the row, enables **Set Address**, and presses **Save**.
5. Observed: the connected wallet receives a `grantRootRoles` transaction addressed to the resolver. Once confirmed, the contractor holds global `ROLE_SET_ADDR` and can replace the ETH address record of any name using that resolver.

### Part B — Deterministic calldata-level PoC

Prerequisites:

* Check out the audited revision `63772fd872af472ced58b009499355f3430c2a86`.
* Use Node.js 22 and pnpm 10, as specified by the competition.
* Install dependencies with `pnpm install --frozen-lockfile`.

Create the file `apps/portal/src/audit-pocs/unresolved-resource-root-escalation.poc.test.ts` in the scoped repository with the following content:

```ts
import { describe, expect, it } from 'vitest'
import { permissionedResolverGrantRootRolesSnippet } from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import { decodeFunctionData, type WalletClient } from 'viem'
import { prepareGrantResolverRolesTransaction } from '@/features/resolver/helpers/grantResolverRoles'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from '@/lib/roles/resolverRoles'

const ACCOUNT = '0x2222222222222222222222222222222222222222'
const CALLER = '0x3333333333333333333333333333333333333333'
const RESOLVER = '0x4444444444444444444444444444444444444444'

const walletClient = {
  account: { address: CALLER },
  chain: { id: 1 },
} as unknown as WalletClient

describe('PoC: unresolved resolver resource falls back to root', () => {
  it('encodes a non-root resource edit as grantRootRoles', () => {
    const staleName = 'moved-away.eth'
    const staleResource = computeNameResource(staleName)

    const grouped = groupRolesByAccount(
      [
        {
          account: ACCOUNT,
          resource: staleResource,
          roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}`,
        },
      ],
      // The role remains, but the resolver no longer lists its name.
      buildResourceToNameMap([]),
    )

    // The original scope is a non-root name resource.
    expect(staleResource).not.toBe(`0x${'00'.repeat(32)}`)
    expect(grouped).toHaveLength(1)
    expect(grouped[0].resolvedNames).toEqual([])

    // This reproduces ResolverRolesSidebar's scope derivation.
    const submittedName =
      grouped[0].resolvedNames.find((name) => name !== '(root)') ?? ''
    expect(submittedName).toBe('')

    // Add ROLE_SET_ADDR to the existing role selection and build the real request.
    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: submittedName,
      account: ACCOUNT,
      roles: [...grouped[0].decodedRoles, 'ROLE_SET_ADDR'],
      walletClient,
      chainId: 1,
    })
    if (intent.request.type !== 'eoa') throw new Error('expected EOA request')

    const decoded = decodeFunctionData({
      abi: permissionedResolverGrantRootRolesSnippet,
      data: intent.request.data,
    })

    // The final wallet calldata grants at resolver root.
    expect(decoded.functionName).toBe('grantRootRoles')
    expect(decoded.args?.[1]).toBe(ACCOUNT)
    expect(intent.request.to).toBe(RESOLVER)
  })
})
```

The test drives the production grouping function, the production transaction builder (`prepareGrantResolverRolesTransaction`), and the exact scope-derivation expression from `ResolverRolesSidebar.tsx:169`, then decodes the final calldata the wallet would receive. Adding `'ROLE_SET_ADDR'` to the selection mirrors the administrator enabling that permission in the edit dialog.

From the repository root, enter the Explorer workspace so Vitest loads its Vite configuration and `@/` alias, then run:

```bash
cd apps/portal
./node_modules/.bin/vitest run src/audit-pocs/unresolved-resource-root-escalation.poc.test.ts
```

On Windows PowerShell, the equivalent verified command is:

```powershell
cd apps/portal
& '.\node_modules\.bin\vitest.CMD' run src/audit-pocs/unresolved-resource-root-escalation.poc.test.ts
```

Observed result:

```
 Test Files  1 passed (1)
      Tests  1 passed (1)
```

Verified against the frozen revision's dependencies with Vitest `v4.1.5`. The decisive assertion is `decoded.functionName === 'grantRootRoles'`: a nonzero name resource (derived from `moved-away.eth`) is absent from the resource-to-name map, and the production builder converts the resulting empty name into resolver-root calldata addressed to the resolver with the target account as grantee.

**Expected result:** an unmapped non-root resource stays identifiable as that resource and is not editable through a name-based form; a resolver-root grant is produced only after the administrator explicitly selects root scope.

**Observed result:** the resource-to-name lookup returns an empty `resolvedNames` array; the sidebar converts the missing mapping into `name: ''`; the production builder interprets the empty string as root; the final wallet request decodes as `grantRootRoles(bitmap, targetAccount)`.
