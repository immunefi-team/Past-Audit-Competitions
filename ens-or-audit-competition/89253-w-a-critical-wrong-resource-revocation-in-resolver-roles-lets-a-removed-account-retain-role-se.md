# 89253 w a critical wrong resource revocation in resolver roles lets a removed account retain role set addr and redirect another ens name

**Submitted on Aug 18th 2026 at 08:58:48 UTC by @ngosytuan for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89253
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

### Summary

The Resolver Roles UI groups all role assignments by account without preserving the role's resource. If the same account has roles on two names handled by one PermissionedResolver, the UI displays a single row containing both names and the union of both sets of roles.

When the resolver owner selects **Remove user**, the confirmation dialog states that the user will be removed "from all roles." However, the application selects only the first non-root name in the grouped row and constructs one name-scoped revocation transaction for that name. The connected wallet therefore signs valid but incomplete calldata:

```
authorizeNameRoles(DNS(firstName), unionOfDisplayedRoles, account, false)
```

The second name resource is not included in that call. The transaction succeeds, the application completes the removal flow, and the removed account retains its roles on every other resource that was folded into the row.

For `ROLE_SET_ADDR`, the former role holder can subsequently change the ETH address of the unaffected name to an attacker-controlled address. Users or applications resolving that ENS name can then send funds to the attacker.

### Root cause

1. [`groupRolesByAccount`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220) uses only the lowercase account as its grouping key. It appends roles and decoded role names from every resource into one group while retaining the resources only as display names.
2. [`ResolverRolesSidebar`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L173) reduces that list to exactly one write target with `resolvedNames.find((n) => n !== '(root)')`.
3. [`handleRemoveUser`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323) passes the chosen name together with the union of all displayed roles, despite the confirmation dialog promising removal "from all roles."
4. The removal mutation invokes [`revokeResolverRoles`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L277) only once.
5. [`prepareRevokeResolverRolesTransaction`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74) treats the selected non-empty name as `scope: 'name'`. ENSJS consequently encodes one `authorizeNameRoles` call for only that DNS name.

This is not caused by stale or incorrect indexed data. The input contains two accurate resource-specific assignments. The application itself destroys the resource-to-role association while preparing the transaction.

### Attack sequence

{% stepper %}
{% step %}

## Shared resolver roles

`alpha.eth` and `beta.eth` use the same PermissionedResolver.

A collaborator has legitimately been granted `ROLE_SET_ADDR` on both names. Later, the name owner needs to revoke that collaborator's access.
{% endstep %}

{% step %}

## Grouped UI row

The portal groups both assignments into one row for the collaborator and displays both names.
{% endstep %}

{% step %}

## Misleading removal confirmation

The owner connects their wallet, selects the row, clicks **Remove user**, and confirms the dialog stating that the collaborator will be removed from all roles.
{% endstep %}

{% step %}

## Incomplete revocation transaction

The portal asks the wallet to approve only `authorizeNameRoles(DNS(alpha.eth), ROLE_SET_ADDR, collaborator, false)`.
{% endstep %}

{% step %}

## Retained role

The transaction succeeds. `ROLE_SET_ADDR` is removed from `alpha.eth`, but it remains active on `beta.eth`.
{% endstep %}

{% step %}

## Redirected resolution

The collaborator uses the retained role to replace `beta.eth`'s ETH address with an attacker-controlled address. Payments made using `beta.eth` resolution can be redirected to the attacker.
{% endstep %}
{% endstepper %}

The same bug is more severe for a row that mixes root and name-scoped roles: the UI may label the row as having global roles, but it selects the non-root name and never invokes the root-scoped revocation path.

## Proof of Concept

The following test uses the real application grouping function and the real `prepareRevokeResolverRolesTransaction` helper backed by the pinned ENSJS dependency. It then decodes the generated calldata and applies the exact single-resource bitmap revocation semantics.

It proves all of the following in one runnable test:

* The two resource-specific assignments become one UI row.
* The UI selects only `alpha.eth`.
* The actual transaction is `authorizeNameRoles` scoped to `alpha.eth`.
* `alpha.eth` is revoked while `beta.eth` retains `ROLE_SET_ADDR`.
* The attacker retains the authorization required to redirect `beta.eth`'s ETH address.

Save this file as:

```
apps/portal/src/features/resolver/components/resolverRolesWrongResource.poc.test.ts
```

```ts
import { parseAbi } from 'viem'
import { decodeFunctionData } from 'viem/utils'
import { packetToBytes } from 'viem/ens'
import { sepolia } from 'viem/chains'
import { toHex } from 'viem'
import { describe, expect, it } from 'vitest'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from '@/lib/roles/resolverRoles'

const VICTIM = '0x1111111111111111111111111111111111111111' as const
const ATTACKER = '0x2222222222222222222222222222222222222222' as const
const RESOLVER = '0x3333333333333333333333333333333333333333' as const
const ALPHA = 'alpha.eth'
const BETA = 'beta.eth'

const authorizeNameRolesAbi = parseAbi([
  'function authorizeNameRoles(bytes name, uint256 roleBitmap, address account, bool grant)',
])

describe('PoC: resolver role removal targets only the first grouped name', () => {
  it('leaves the attacker able to set the ETH address on the second name', () => {
    const alphaResource = computeNameResource(ALPHA)
    const betaResource = computeNameResource(BETA)

    // This is valid indexed state: the same account holds ROLE_SET_ADDR on two
    // different name resources of one PermissionedResolver.
    const grouped = groupRolesByAccount(
      [
        {
          account: ATTACKER,
          resource: alphaResource,
          roleBitmap: resolverRoles.ROLE_SET_ADDR.toString(),
        },
        {
          account: ATTACKER,
          resource: betaResource,
          roleBitmap: resolverRoles.ROLE_SET_ADDR.toString(),
        },
      ],
      buildResourceToNameMap([{ name: ALPHA }, { name: BETA }]),
    )

    expect(grouped).toHaveLength(1)
    const row = grouped[0]!

    // Exact selection logic in ResolverRolesSidebar: all resources were lost
    // during grouping and only the first non-root name is kept for the write.
    const roleName = row.resolvedNames.find((name) => name !== '(root)') ?? ''

    // Exact app helper used by handleRemoveUser/removeUserMutation. A minimal
    // wallet client is sufficient because preparation is a pure encode.
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: ATTACKER,
      roles: row.decodedRoles as Array<keyof typeof resolverRoles>,
      walletClient: {
        account: { address: VICTIM },
        chain: sepolia,
      } as never,
      chainId: sepolia.id,
    })

    const decoded = decodeFunctionData({
      abi: authorizeNameRolesAbi,
      data: intent.request.data,
    })

    expect(row.resolvedNames).toEqual([ALPHA, BETA])
    expect(roleName).toBe(ALPHA)
    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect(decoded.args).toEqual([
      toHex(packetToBytes(ALPHA)),
      resolverRoles.ROLE_SET_ADDR,
      ATTACKER,
      false,
    ])

    // PermissionedResolver.authorizeNameRoles derives exactly one resource
    // from the single DNS name above. _revokeRoles clears that resource only.
    const before = new Map([
      [alphaResource, resolverRoles.ROLE_SET_ADDR],
      [betaResource, resolverRoles.ROLE_SET_ADDR],
    ])
    const after = new Map(before)
    after.set(
      computeNameResource(roleName),
      (after.get(computeNameResource(roleName)) ?? 0n) &
        ~resolverRoles.ROLE_SET_ADDR,
    )

    expect(after.get(alphaResource)).toBe(0n)
    expect(after.get(betaResource)).toBe(resolverRoles.ROLE_SET_ADDR)

    // Security impact: after the victim signs the UI's successful "Remove
    // user ... from all roles" transaction, the attacker still has the exact
    // role needed to redirect beta.eth's ETH address record.
    expect(
      (after.get(betaResource) ?? 0n) & resolverRoles.ROLE_SET_ADDR,
    ).toBe(resolverRoles.ROLE_SET_ADDR)

    console.log(
      [
        `UI grouped names: ${row.resolvedNames.join(', ')}`,
        `UI-selected transaction scope: ${roleName}`,
        `Encoded function: ${decoded.functionName}`,
        `alpha.eth ROLE_SET_ADDR after revoke: ${after.get(alphaResource)}`,
        `beta.eth ROLE_SET_ADDR after revoke: ${after.get(betaResource)}`,
        'Impact: attacker still controls beta.eth address resolution',
      ].join('\n'),
    )
  })
})
```

### Run

```bash
CI=1 node_modules/.bin/vitest run src/features/resolver/components/resolverRolesWrongResource.poc.test.ts --reporter=verbose --environment=node
```

Expected security-relevant output:

```
UI grouped names: alpha.eth, beta.eth
UI-selected transaction scope: alpha.eth
Encoded function: authorizeNameRoles
alpha.eth ROLE_SET_ADDR after revoke: 0
beta.eth ROLE_SET_ADDR after revoke: 1
Impact: attacker still controls beta.eth address resolution

Test Files  1 passed (1)
     Tests  1 passed (1)
```

The test exits with code `0`. No public ENS name or deployed contract is modified.

## Recommended remediation

{% stepper %}
{% step %}

## Preserve resource boundaries

Group rows by `(account, resource)` instead of account alone, or retain a `resource -> roles` mapping in every grouped row.
{% endstep %}

{% step %}

## Revoke every affected resource

For a UI action that promises to remove all roles, build one revocation for every affected resource. Use the root-scoped call for root roles and one name-scoped call per name resource.
{% endstep %}

{% step %}

## Validate transaction coverage

Before requesting a wallet signature, assert that the generated transaction set covers every resource and role shown by the confirmation dialog.
{% endstep %}

{% step %}

## Add regression tests

Add regression tests for two names sharing one resolver and for mixed root/name-scoped assignments.
{% endstep %}
{% endstepper %}

## Why this is not a listed known issue

This finding does not rely on stale indexer data, wallet account switching, an incorrect `from` field, the documented seven-day HCA session model, or any transaction-manager funding/policy issue. Correct resource-specific role data enters the portal, but `groupRolesByAccount` removes the resource association and the sidebar deterministically signs a revocation for the wrong subset of resources.
