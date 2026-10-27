# 92859 w a critical unresolved name scoped resolver role is converted into a root grant enabling unauthorized record changes for unrelated names

## #92859 \[W\&A-Critical] Unresolved name-scoped resolver role is converted into a root grant, enabling unauthorized record changes for unrelated names

**Submitted on Sep 12th 2026 at 21:26:17 UTC by @andro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92859
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

### Description

### Brief/Intro

When ENS Explorer cannot map an existing non-root resolver resource hash back to a name, it represents the missing mapping as an empty `resolvedNames` array. The role editor then converts that condition to `name === ''`, which the transaction builder reserves for resolver-root authority. As a result, adding a permission to an unresolved name-scoped row calls `grantRootRoles` and grants global authority instead of preserving or rejecting the original name scope.

If `ROLE_SET_ADDR` is added, the recipient can modify address records for unrelated names on the same resolver. This can redirect payments sent through those names.

### Vulnerability Details

Resolver role entries identify their scope with a resource hash. `groupRolesByAccount()` attempts to recover a display name from `resourceToName`. If no mapping exists, it retains the account and permissions but drops the unresolved resource from the editable model:

```ts
const resolvedName = resourceToName?.get(role.resource.toLowerCase())

// ...

if (resolvedName) names.add(resolvedName)
grouped.set(account, {
  account,
  resolvedNames: names,
  roles: [role],
  decodedRoles: [...decoded],
})
```

Affected code: `apps/portal/src/lib/roles/resolverRoles.ts`, lines 197-212 at the audited revision.

An unresolved non-root resource is therefore returned with `resolvedNames: []`. The edit sidebar remains available and derives its write scope as follows:

```ts
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

Affected code: `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx`, lines 167-169.

When the administrator saves, the sidebar passes this empty `roleName` to the grant helper. The helper gives the empty string privileged meaning:

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
    }
```

Affected code: `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts`, lines 47-61.

The failed lookup and an intentional selection of resolver root consequently reach the transaction layer as the same value. The final wallet request calls `grantRootRoles` even though the stored role resource is a nonzero name resource.

### How the Unresolved State Can Occur

A valid name-scoped role can outlive the resolver's current name list. For example:

1. `moved-away.eth` uses a shared permissioned resolver.
2. A contractor receives a role scoped to `moved-away.eth`.
3. The name later changes to a different resolver, while the old resolver role entry remains.
4. The old resolver no longer supplies the name needed to build the resource-to-name map.
5. The role entry is still grouped by account, but its resource cannot be rendered as a name.

No malformed on-chain state is required. The vulnerable state follows from ordinary resolver migration combined with a remaining role entry.

### Root Cause

The application discards an unresolved non-root resource identifier and uses an empty string both for lookup failure and explicit resolver-root scope. The transaction boundary accepts this ambiguous value and fails open to the most privileged scope.

### Attack Scenario

1. An organization manages several production names through one permissioned resolver.
2. An attacker-controlled contractor account has `ROLE_SET_TEXT` only for `moved-away.eth`.
3. `moved-away.eth` changes resolver, but the contractor's old name-scoped role remains on the original resolver.
4. An administrator reviews users on the original resolver. The contractor's account remains visible because its role entry still exists, while its associated name cannot be resolved.
5. The administrator opens that row and enables **Set Address**, intending to update the limited account represented by the row.
6. The application supplies `name: ''` and constructs `grantRootRoles(..., contractor)`.
7. After the administrator signs the request, the contractor has global `ROLE_SET_ADDR` on the original resolver.
8. The contractor changes the ETH address of an unrelated production name on that resolver to an attacker-controlled address. Payments resolved through that name are redirected.

### Impact Details

The transaction grants permissions over all current and future names using the permissioned resolver rather than one name. With `ROLE_SET_ADDR`, the target account can replace cryptocurrency address records for unrelated names. Users relying on those ENS records can then send assets to an attacker-controlled address.

The scope substitution occurs between the administrative row the victim selected and the calldata submitted to the already-connected wallet. It is deterministic whenever the row contains only unresolved non-root resources. The maximum blast radius is every name using the affected resolver.

## Risk Breakdown

* **Victim:** an authorized resolver administrator using the normal Explorer role-management interface.
* **Attacker prerequisite:** the attacker's account has a stale name-scoped role whose resource is no longer present in the resolver's current resource-to-name map.
* **Required victim interaction:** the administrator adds at least one permission to that visible account row and signs the generated transaction.
* **Complexity:** low after the prerequisite state exists; the vulnerable conversion is deterministic.
* **Privileges gained:** the newly selected permissions are granted at resolver root.
* **Security boundary crossed:** authority changes from one name resource to every name on the resolver.
* **Maximum consequence:** address-record replacement for unrelated names and redirection of payments.

### References

* ENS Audit Competition scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* ENS Audit Competition information and known issues: <https://immunefi.com/audit-competition/audit-competition-ens/information/>

### Proof of Concept

### Proof of Concept

### Prerequisites

* Using audited commit `63772fd872af472ced58b009499355f3430c2a86`.
* Use Node.js 22 and pnpm 10.
* Install dependencies with `pnpm install --frozen-lockfile`.

### PoC File

Create `apps/portal/src/audit-pocs/unresolved-resource-root-escalation.poc.test.ts` with the following code:

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

### Run Command

From the repository root, enter the Explorer workspace so Vitest loads its Vite configuration and `@/` alias:

```bash
cd apps/portal
./node_modules/.bin/vitest run src/audit-pocs/unresolved-resource-root-escalation.poc.test.ts
```

### Expected Output

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

The decisive assertion is `decoded.functionName === 'grantRootRoles'`: a nonzero resource derived from `moved-away.eth` is absent from the name map, and the production transaction builder converts the resulting empty name into resolver-root calldata.

## Recommendation

Preserve the resource hash in the editable row and model scope explicitly, for example:

```ts
type ResolverRoleScope =
  | { type: 'root' }
  | { type: 'name'; name: string }
  | { type: 'unresolved'; resource: Hex }
```

Only `{ type: 'root' }` should be allowed to produce `grantRootRoles`. If a non-root resource cannot be mapped to a name, display its resource hash, make the name-based editor read-only, and require an explicit resource-aware cleanup or recovery flow. Reject empty or unresolved scopes before opening the wallet.

Add a regression test that supplies a nonzero resource absent from `resourceToName` and asserts that no grant transaction can be prepared. Also require explicit confirmation showing **Root (all names)** before any root grant.

## Known-Issue and Duplicate Analysis

The published QA-01 issue concerns functional problems when role combinations are edited. This report demonstrates a separate privilege-boundary failure: lookup failure for a non-root resource is converted into resolver-root authority. The affected data transition and required remediation are specific to unresolved resources.
