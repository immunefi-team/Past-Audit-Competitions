# 92752 w a critical removing a resolver user revokes only one resource and leaves root or other name authority active

**Submitted on Sep 12th 2026 at 00:29:46 UTC by @andro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92752
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user funds

## Description

## Brief Intro

The resolver-user removal flow promises to remove a user from all roles, but it revokes only one name-scoped resource from a row that may combine resolver-root and multiple name resources. A removed or compromised account can therefore retain address-setting authority and redirect payments.

## Vulnerability Details

The resolver-user removal flow promises to remove a user from all roles, but it revokes only one name-scoped resource even when the displayed user row represents roles belonging to multiple resources.

The application groups resolver permissions by account instead of preserving each (account, resource) pair independently.

As a result, roles from resolver root and multiple ENS names can be merged into one displayed row.

The data layer groups resolver roles by `account` alone. In `apps/portal/src/lib/roles/resolverRoles.ts`, `groupRolesByAccount` merges every role entry of the same account into one row, concatenating the decoded role bitmaps while keeping the resources only as a display array:

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

The edit sidebar (`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx`, lines 165–169) then discards the multi-resource information and derives a single write scope from the display names:

```ts
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The remove handler (lines 307–323) submits that single derived name together with the union of every merged role:

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

Finally, the transaction helper (`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts`, lines 45–61) interprets a non-empty `name` as exactly one name-scoped resource. It has no way to revoke the original role entries at their original resources:

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

The transaction succeeds, but the roles belonging to resolver root and every other name remain active on-chain. The interface reports a successful security-sensitive removal while the account retains authority.

## Preconditions

* One resolver account has active grants on at least two resources, such as resolver root and one named node. This state is creatable entirely inside the application: the **Add user** flow can grant an account root-scoped roles and, in a second operation, grant the same account a name-scoped role; both grants then render as one merged row. Direct on-chain grants produce the same row. The sidebar's own label for such rows ("Global roles (all names)") confirms mixed rows are a supported display state.
* An administrator with authority to revoke those grants uses the combined-row **Remove user** action. The application gates this management page on the connected account holding the root `Set Address` role, and under that same authority model the name-scoped revocation it builds succeeds on-chain.
* The removed account is compromised, malicious, or no longer trusted and later exercises the retained role.

## Attack Path

1. An organization uses one permissioned resolver for several names.
2. A service account is granted `ROLE_SET_ADDR` at resolver root and a harmless text role on `victim.eth` (both supported by the application's **Add user** flow).
3. The service account is compromised or offboarded.
4. An administrator opens its single combined row. The page states that it has global roles and offers **Remove user**.
5. The administrator confirms removal from all roles and signs the transaction presented by the application.
6. The transaction succeeds, but it targets only `victim.eth`. The global `ROLE_SET_ADDR` grant remains.
7. The removed account later changes the ETH address of `victim.eth` (or any name covered by the retained root grant) to an attacker-controlled address.
8. A payer resolves `victim.eth` and transfers funds. The funds go to the attacker.

No race, compromised frontend, unusual wallet, or idealized contract behaviour is required. It uses supported root and name role grants and the normal removal flow.

## Likelihood and Risk Breakdown

* **Attacker prerequisite:** the attacker previously received an ordinary delegated resolver role, or compromised such an account.
* **Victim interaction:** the administrator uses the normal **Remove user** confirmation and signs the transaction presented by the application.
* **Reliability:** deterministic whenever one displayed account has roles on more than one resource.
* **Blast radius:** every name covered by a retained root role, plus every additional name resource omitted from the revocation.
* **Why this is security-critical:** the application reports successful removal from all roles while the contract still authorizes payment-address changes.

## Impact

A compromised or offboarded delegate can retain global `ROLE_SET_ADDR` after an administrator uses the advertised remove-all control. The retained contract authority permits changing payment addresses for every name using the resolver, which can redirect later transfers to the attacker.

## Severity Justification

Critical maps to the program's direct-theft impact: the surviving address-setting authority can deterministically substitute an attacker payment address after the administrator believes the delegate was removed. The immediate bug is the failed security revocation; theft is the direct exercise of the retained role and does not require another privilege escalation.

Three properties close the usual downgrade arguments:

* **The wallet confirmation cannot inform the admin.** The application's own dialogue promises "remove this user from all roles"; the wallet then displays ABI-encoded calldata with a DNS-encoded name hash. Decoding function arguments is not the standard applied to administrators using a management UI, and the application is the layer responsible for building the correct scope.
* **Silence is the realistic outcome.** In the normal case where the administrator manages grants made through this UI, the name-scoped revocation succeeds and nothing in the interface indicates that root authority remains. (If an administrator lacked authority at the name resource, the visible transaction would revert loudly — but either way the root grant is never revoked.)
* **Later discovery does not undo the impact.** The account's row remains visible after a refresh because roles remain, but the attacker can exercise the retained authority in the same block the partial revocation lands; payment redirections already performed are not undone by the admin noticing the row later.

## Duplicate and Prior-Issue Analysis

The published QA-01 entry describes functional errors from certain role combinations and expressly states that a demonstrated security consequence — "a role combination that grants unintended authority" — is a new finding. This report proves retained root authority in the final calldata of the removal flow, which is that carve-out. None of the other published R2, R3, SEC, EXP, WEB, or QA entries describes grouping resolver resources into one row and revoking only one resource.

This finding is distinct from a report on unresolved resource rows escalating to root grants on save: that path fails open from an unknown resource into a new root grant during an edit, while this path silently drops scope boundaries during a revocation. The two share only the sidebar's display layer and are independently remediable.

## References

* Competition scope and impacts: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Competition published issues: <https://immunefi.com/audit-competition/audit-competition-ens/information/>

## Proof of Concept

## Proof of Concept

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

The only wallet request produced by the advertised remove-all action is `authorizeNameRoles(name, mergedBitmap, account, false)`. The same bytes cannot be decoded as `revokeRootRoles`, so no root revocation is ever submitted, and the account's root grant survives the flow that the application reports as complete removal.

The PoC uses:

* The production `groupRolesByAccount` function.
* The production `prepareRevokeResolverRolesTransaction` transaction builder.
* The same resource-selection behavior used by `ResolverRolesSidebar.tsx`.
* Decoding of the final calldata presented to the wallet.

The resulting wallet request is:

```
authorizeNameRoles(name, mergedBitmap, account, false)
```

The same calldata cannot be decoded as a root-role revocation.

Therefore, no resolver-root revocation is submitted.

The root grant survives the application's advertised **"remove from all roles"** operation.

## Expected Result

Confirming **Remove user from all roles** should revoke every displayed permission at its original resource.

If an account has permissions at:

* Resolver root.
* `victim.eth`.
* Another ENS name.

The application should submit the required revocation for each corresponding resource.

Alternatively, the UI should clearly restrict the removal operation to one explicitly identified resource.

## Observed Result

The application merges the permissions into one row and reduces that row to the first non-root name.

The final wallet request contains only:

```
authorizeNameRoles(name, mergedBitmap, account, false)
```

* No root-role revocation is generated.
* The account therefore retains its resolver-root permissions.
* Other name-scoped permissions may also remain active.
* The UI nevertheless reports the removal as successful.

## Recommended Fix

Preserve each `(account, resource)` pair as an independent permission entry.

Do not merge permissions belonging to multiple resources into a single editable security object unless the application also retains the exact original resource boundaries.

### Possible fixes include:

* Display one editable row for every `(account, resource)` pair.
* Preserve every original resolver-role entry internally and submit one revocation transaction for each original resource represented by the row.

Construct the transaction scope directly from the original resource identifier rather than from a human-readable display-name array.

After the removal transaction completes, read the affected permissions back from the chain. Report success only if every permission the administrator intended to remove has actually been cleared.

### Add a regression test that:

* Creates an account with root-scoped `ROLE_SET_ADDR`.
* Adds name-scoped `ROLE_SET_TEXT`.
* Removes the combined account through the UI flow.
* Captures all generated requests.
* Confirms that both the root revocation and the name-scoped revocation are present.
