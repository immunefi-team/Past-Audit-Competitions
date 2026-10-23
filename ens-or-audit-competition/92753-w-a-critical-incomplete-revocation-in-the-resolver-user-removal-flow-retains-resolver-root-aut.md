# 92753 w a critical incomplete revocation in the resolver user removal flow retains resolver root authority leading to theft of user funds via payment address redirection

**Submitted on Sep 12th 2026 at 00:31:46 UTC by @Kodekloner43926 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92753
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user funds

## Description

## Brief/Intro

The ENS Explorer's resolver user-management page merges every role grant of an account into a single table row, and its **Remove user** action promises to "remove this user from all roles". The action actually revokes only one name-scoped resource from that row, so a removed account silently retains its resolver-root grants, including `ROLE_SET_ADDR`. A compromised or offboarded account can then change the ETH payment address of any name served by the shared resolver and receive funds that payers send to the affected name.

## Vulnerability Details

Affected asset: Explorer app (`apps/portal`), competition frozen revision `63772fd872af472ced58b009499355f3430c2a86`. The vulnerable flow is the resolver roles management page (`apps/portal/src/routes/resolver/$address/roles/`), used by administrators of a Permissioned Resolver.

**The data layer collapses multiple resources into one row.** In `apps/portal/src/lib/roles/resolverRoles.ts`, `groupRolesByAccount` groups role records by `account` alone, concatenating the decoded role bitmaps and keeping the resources only as a display array:

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
    /* ... new group with the same fields ... */
  }
}
```

An account with a resolver-root grant and a name-scoped grant therefore renders as one merged row (the UI itself labels such rows "Global roles (all names)").

**The edit sidebar reduces that row to a single scope.** In `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx` (lines 165–169):

```ts
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`roleName` is the first non-root display name — one single name chosen for the whole merged row.

**The remove handler submits that one name together with the union of every merged role** (lines 307–323):

```ts
const handleRemoveUser = () => {
  if (
    !selectedAccount ||
    decodedRoles.length === 0 ||
    removeUserMutation.isPending
  )
    return

  removeUserMutation.reset()
  setPendingAction({
    type: 'remove',
    name: roleName,
    account: selectedAccount,
    roles: decodedRoles,
  })
  openModal()
}
```

The confirmation dialog shown to the administrator states: *"Are you sure you want to remove this user from all roles? This action cannot be undone."*

**The transaction helper can only revoke at one scope.** In `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts` (lines 45–61), a non-empty `name` selects exactly one name-scoped revocation; there is no path that revokes the account's other resources:

```ts
const writeParams = revokeResolverRolesWriteParameters(
  walletClient as Parameters<typeof revokeResolverRolesWriteParameters>[0],
  name === ''
    ? {
        resolverAddress,
        targetAccount: account,
        scope: 'root',
        roles: roles as ResolverRole[],
      }
    : {
        resolverAddress,
        targetAccount: account,
        scope: 'name',
        name,
        roles: roles as ResolverRole[],
      },
)
```

For a non-empty `name`, ensjs's `revokeResolverRolesWriteParameters` emits `authorizeNameRoles(name, bitmap, account, false)` on the resolver — a revocation bound to that one name's resource. The resolver-root grant (stored at the root resource) and any other name's grant are never touched, yet the interface reports a successful removal from all roles.

**Preconditions** (all creatable through the application itself):

* The account holds grants on at least two resources, e.g. resolver-root `ROLE_SET_ADDR` plus a name-scoped role. The **Add user** flow supports granting root-scoped roles and name-scoped roles to the same account (two operations), which then render as one merged row; direct on-chain grants produce the same row.
* An administrator with revocation authority uses the merged row's **Remove user** action. The page is gated on the connected account holding the resolver-root `Set Address` role, and under that same authority model the name-scoped revocation it builds succeeds on-chain.
* The removed account is compromised, malicious, or no longer trusted, and later exercises the retained role.

**Relation to published known issues.** The published QA-01 entry covers functional errors from certain role combinations and expressly carves out demonstrated security consequences — "a role combination that grants unintended authority" is stated there to be a new finding. This report demonstrates retained root authority in the final calldata of the removal flow, which is that carve-out. No other published R2, R3, SEC, EXP, WEB, or QA entry describes grouping resolver resources into one row and revoking only one resource. This finding is also distinct from a report on unresolved resource rows being escalated to root grants on save: that path fails open from an unknown resource into a new root grant during an edit, while this path silently drops scope boundaries during a revocation — different trigger, different direction, independently remediable.

**Recommended fix.** Represent each `(account, resource)` pair as a separate editable row, or retain every original role entry and submit one revocation per original resource, building the transaction scope from the role entry's resource identifier rather than a display-name array. After confirmation, read every affected resource on-chain and report success only when all intended bitmaps are cleared. As a regression test: create one account with a root `ROLE_SET_ADDR` grant and a name-scoped `ROLE_SET_TEXT` grant, remove the account through the merged-row action, capture every emitted request, and assert that both a root revocation and the name revocation are present.

## Impact Details

Selected impact: **Critical — Direct theft of user funds.**

Attack path:

1. An organization uses one permissioned resolver for several names.
2. A service account is granted `ROLE_SET_ADDR` at resolver root and a harmless text role on `victim.eth` (both supported by the application's **Add user** flow).
3. The service account is compromised or offboarded.
4. An administrator opens its single merged row (labeled "Global roles (all names)") and confirms **Remove user**.
5. The administrator signs the transaction the application presents; it succeeds, but targets only `victim.eth`. The global `ROLE_SET_ADDR` grant remains.
6. The removed account changes the ETH address of `victim.eth` — or any name covered by the retained root grant — to an attacker-controlled address.
7. A payer resolves the name and transfers funds. The funds go to the attacker.

No race, compromised frontend, unusual wallet, or idealized contract behavior is required — only supported root and name role grants and the normal removal flow.

Risk breakdown:

* **Attacker prerequisite:** previously received (or compromised) an ordinary delegated resolver role.
* **Victim interaction:** the administrator uses the normal remove-all confirmation and signs the transaction the application presents.
* **Reliability:** deterministic whenever one displayed account has roles on more than one resource.
* **Blast radius:** every name covered by a retained root grant, plus every additional name resource omitted from the revocation. Funds at risk: any payment sent to a name on the resolver after the attacker redirects its address record.

Why this is Critical rather than a lower severity:

* **The wallet confirmation cannot inform the admin.** The application's own dialog promises "remove this user from all roles"; the wallet then displays ABI-encoded calldata with a DNS-encoded name hash. Decoding function arguments is not the standard applied to administrators using a management UI, and the application is the layer responsible for building the correct scope.
* **Silence is the realistic outcome.** In the normal case (grants made through this UI), the name-scoped revocation succeeds and nothing in the interface indicates that root authority remains. (If an administrator lacked authority at the name resource, the visible transaction would revert loudly — but either way the root grant is never revoked.)
* **Later discovery does not undo the impact.** The attacker can exercise the retained authority in the same block the partial revocation lands; payment redirections already performed are not undone by the admin noticing the row later.

## References

* ENS Audit Competition scope and impacts: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* ENS Audit Competition published known issues: <https://immunefi.com/audit-competition/audit-competition-ens/information/>
* Affected files (frozen revision `63772fd872af472ced58b009499355f3430c2a86`): `apps/portal/src/lib/roles/resolverRoles.ts`, `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx`, `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts`

## Proof of Concept

## Proof of Concept

### Part A — Step-by-step (application flow)

1. Open the management page of a Permissioned Resolver in the Explorer as an administrator (an account holding the resolver-root `Set Address` role): `/<resolver-address>/roles`.
2. Using **Add user**, grant a service account the `Set Address` role at **Root (all nodes)** scope.
3. Using **Add user** again, grant the same account the `Set Text` role at `victim.eth` scope. The table now shows one merged row for that account, labeled "Global roles (all names)".
4. Open the merged row and press **Remove user**. The dialog asks: "Are you sure you want to remove this user from all roles? This action cannot be undone." Confirm and sign.
5. Observed: the transaction succeeds and the UI reports the user removed. On-chain, the account still holds the resolver-root `ROLE_SET_ADDR` grant (a direct `hasRoles`/`getRoles` read against the resolver's root resource still returns the role), so the account can still set address records for every name using the resolver.

### Part B — Deterministic calldata-level PoC

Prerequisite: install the frozen revision's dependencies with `pnpm install`.

Create the file `apps/portal/src/audit-pocs/resolver-role-removal.poc.test.ts` in the scoped repository with the following content:

```ts
import { describe, expect, it } from 'vitest'
import {
  permissionedResolverAuthorizeNameRolesSnippet,
  permissionedResolverRevokeRootRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import { decodeFunctionData, type WalletClient } from 'viem'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from '@/lib/roles/resolverRoles'

const ACCOUNT = '0x1111111111111111111111111111111111111111'
const CALLER = '0x3333333333333333333333333333333333333333'
const RESOLVER = '0x4444444444444444444444444444444444444444'
const ROOT_RESOURCE = `0x${'00'.repeat(32)}`

const walletClient = {
  account: { address: CALLER },
  chain: { id: 1 },
} as unknown as WalletClient

describe('PoC: resolver role removal loses resource boundaries', () => {
  it('selects one name even though the row also contains a root-scoped role', () => {
    const name = 'victim.eth'
    const grouped = groupRolesByAccount(
      [
        {
          account: ACCOUNT,
          resource: ROOT_RESOURCE,
          roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}`,
        },
        {
          account: ACCOUNT,
          resource: computeNameResource(name),
          roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}`,
        },
      ],
      buildResourceToNameMap([{ name }] as never),
    )

    expect(grouped).toHaveLength(1)
    expect(grouped[0].resolvedNames).toEqual(['(root)', name])

    // ResolverRolesSidebar.tsx:169 chooses the first non-root name and lines
    // 317-320 attach every merged role to that single scope for "Remove user".
    const roleName = grouped[0].resolvedNames.find((n) => n !== '(root)') ?? ''
    expect(roleName).toBe(name)
    expect(grouped[0].decodedRoles).toContain('ROLE_SET_ADDR')

    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: ACCOUNT,
      roles: grouped[0].decodedRoles,
      walletClient,
      chainId: 1,
    })
    if (intent.request.type !== 'eoa') throw new Error('expected EOA request')

    const decodedNameWrite = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data: intent.request.data,
    })
    expect(decodedNameWrite.functionName).toBe('authorizeNameRoles')
    expect(decodedNameWrite.args?.[2]).toBe(ACCOUNT)
    expect(decodedNameWrite.args?.[3]).toBe(false)

    // A correct remove-all flow also needs a root revocation. The only emitted
    // request is name-scoped and cannot even be decoded as revokeRootRoles.
    expect(() =>
      decodeFunctionData({
        abi: permissionedResolverRevokeRootRolesSnippet,
        data: intent.request.data,
      }),
    ).toThrow()
  })
})
```

The test uses the production grouping function, the production transaction builder (`prepareRevokeResolverRolesTransaction`), and the exact scope-derivation expression from `ResolverRolesSidebar.tsx:169`, then decodes the final calldata the wallet would receive.

Run from `apps/portal`:

```bash
pnpm test src/audit-pocs/resolver-role-removal.poc.test.ts
```

(Equivalent from the repository root: `pnpm --filter portal test src/audit-pocs/resolver-role-removal.poc.test.ts`.)

Observed result:

```
 Test Files  1 passed (1)
      Tests  1 passed (1)
```

The only wallet request produced by the advertised remove-all action decodes as `authorizeNameRoles(name, mergedBitmap, account, false)`. The same bytes cannot be decoded as `revokeRootRoles`, so no root revocation is ever submitted, and the account's root grant survives the flow that the application reports as complete removal.

**Expected result:** confirming **Remove user from all roles** revokes every displayed role at its original resource, including resolver root and every individual name resource; the flow either submits all required revocations or clearly restricts the action to one identified resource.

**Observed result:** the merged row is reduced to the first non-root name; the final wallet request decodes only as `authorizeNameRoles(name, mergedBitmap, account, false)`; no `revokeRootRoles` request is produced; the account retains its root and other-name grants even though the removal succeeds in the UI.
