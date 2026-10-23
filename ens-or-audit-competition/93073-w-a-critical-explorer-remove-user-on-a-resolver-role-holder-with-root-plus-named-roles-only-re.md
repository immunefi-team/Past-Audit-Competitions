# 93073 w a critical explorer remove user on a resolver role holder with root plus named roles only revokes one named scope leaving the root role active

**Submitted on Sep 14th 2026 at 06:47:29 UTC by @abarbatei for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93073
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer resolver-roles page groups every role an account holds on a PermissionedResolver into one row, discarding which resource each role was granted on. When an account holds a role at the root resource (which the UI labels "Global roles (all names)") and also a role on one or more specific names, the sidebar picks the first non-root name as the scope and sends the union of all roles to the named-scope encoder. "Remove user" therefore submits `authorizeNameRoles(dns(firstName), unionedBitmap, account, false)` and never calls `revokeRootRoles`. The administrator approves a transaction whose arguments do not match the "remove this user" action the UI describes, and the removed account keeps its root role, for example the ability to set address records on every name served by that resolver. "Save changes" has the same scope loss.

## Vulnerability Details

Intended behavior: a PermissionedResolver keeps roles per `(resource, account)`. The root resource grants a role over all names; a name resource grants it for that name only. The Explorer's add-user flow exposes both scopes, and its transaction helpers distinguish them: an empty name means root and encodes `revokeRootRoles`, a non-empty name encodes `authorizeNameRoles(..., false)`. Removing a user should revoke every scope the account holds.

Root cause: the grouping step erases the resource association and the sidebar reconstructs a single scope from it.

1. [`resolverRoles.ts::groupRolesByAccount` lines 193-215](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/lib/roles/resolverRoles.ts#L193-L215) merges every role of an account into one row: `decodedRoles` is the concatenation of all decoded bitmaps and `resolvedNames` a set of names, with no link between a role and its resource.

```ts
    if (existing) {
      existing.roles.push(role)
      existing.decodedRoles.push(...decoded)
      if (resolvedName) existing.resolvedNames.add(resolvedName)
    }
```

2. [`ResolverRolesSidebar.tsx::ResolverRolesSidebar` lines 166-169](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L166-L169) derives one scope for the whole row: the first name that is not `(root)`, or the empty string.

```tsx
  const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
  const resolvedNames = row?.original.resolvedNames ?? []

  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

With `resolvedNames = ['(root)', 'alice.eth', 'bob.eth']` this is `'alice.eth'`. The root scope is only selected when the account holds nothing else.

3. Meanwhile [`ResolverRolesSidebar.tsx::ResolverRolesSidebar` lines 393-395](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L393-L395) tells the administrator the row is global:

```tsx
                    {resolvedNames.includes('(root)')
                      ? 'Global roles (all names)'
                      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
```

4. [`ResolverRolesSidebar.tsx::ResolverRolesSidebar` lines 307-323](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323) stores `{ name: roleName, roles: decodedRoles }` as the pending remove action; the save path at lines 295-305 stores `{ name: roleName, rolesToGrant, rolesToRevoke }` computed from the same unioned list.
5. [`revokeResolverRoles.ts::prepareRevokeResolverRolesTransaction` lines 45-61](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L45-L61) maps `name === ''` to `scope: 'root'` and anything else to `scope: 'name'`. ensjs then encodes `revokeRootRoles(bitmap, account)` for root and `authorizeNameRoles(dnsName, bitmap, account, false)` for a name. With `name = 'alice.eth'` the root scope is never touched.

```ts
    name === ''
      ? { resolverAddress, targetAccount: account, scope: 'root', roles }
      : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

6. The configuration is reachable through the product's own UI: [`ResolverAddUserSheet.tsx::ResolverAddUserSheet` lines 248-285](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L248-L285) offers "Root (all nodes)" and each individual name as a scope, so an administrator who grants a delegate a root role and later a name-specific role produces exactly this row.

Failure path:

1. A resolver administrator grants account T `ROLE_SET_ADDR` at root and `ROLE_SET_ADDR` on `alice.eth` (and, say, `ROLE_SET_TEXT` on `bob.eth`).
2. Later the administrator opens the resolver roles page, selects T, reads "Global roles (all names)", presses "Remove user" and confirms.
3. The wallet receives `authorizeNameRoles(dns("alice.eth"), ROLE_SET_ADDR | ROLE_SET_TEXT, T, false)`. Whether the resolver treats the extra `ROLE_SET_TEXT` bit as a no-op or reverts, the root grant is not part of the call.
4. T still holds `ROLE_SET_ADDR` at root and `ROLE_SET_TEXT` on `bob.eth`. The UI, after refetch, shows T still present, but an administrator who trusted the confirmation has no reason to look again.

The same happens on "Save changes": unchecking a permission produces a revoke for the first named scope only, even when the permission is held at root.

## Impact Details

* The administrator is shown "Global roles (all names)" and a "Remove user" action, then approves calldata that targets a single name. The transaction's scope and role arguments differ from what the UI represents, which is the program's "showing the wrong transaction detail on a transaction the user then approves" concern.
* The practical consequence is a failed revocation. A delegate who was supposed to be removed keeps whatever it held at root. With `ROLE_SET_ADDR` at root that is the ability to change the ETH (and any other coin) address of every name that uses this resolver, so funds sent to those names can be redirected by an account the owner believes is gone. `ROLE_SET_TEXT`, `ROLE_SET_CONTENTHASH` and the other root roles keep equivalent authority over profile data and content hashes.
* Roles on the second and later named resources (`bob.eth` above) are also never revoked.
* No attacker input is needed to trigger the mis-encoding, but the beneficiary is an untrusted third party: the delegate being offboarded. The precondition is only that the same account was granted both a root role and a name role on the same resolver, which the Explorer's own add-user sheet supports.

This is reported as High: it is a silent access-control revocation failure whose retained privilege includes rewriting the address records of every name on the resolver. It is not Critical because the app does not itself redirect funds or hand authority to an arbitrary attacker; it fails to remove authority from a specific previously-trusted account.

## Mitigation

Preserve the resource association from query to transaction.

* In `groupRolesByAccount`, keep `roles` grouped per resource (for example `scopes: Array<{ resource, name, decodedRoles }>`) instead of a flat `decodedRoles` union.
* In `ResolverRolesSidebar`, build one revoke call per scope the account holds: `scope: 'root'` with only the root bitmap, and `scope: 'name'` with that name's bitmap for each named resource. Submit them as a batch or sequentially, and show the list of scopes in the confirmation dialog before opening the wallet.
* Apply the same per-scope diff to "Save changes" so grants and revokes are computed against the scope where the role is actually held.
* Fail closed: if a resource cannot be resolved to a name, disable editing for that account instead of silently dropping it.
* Add a unit test with root + two named resources that asserts the encoded calls for "Remove user" include `revokeRootRoles` for the root bitmap and one `authorizeNameRoles(..., false)` per named resource.

## References

* [`resolverRoles.ts::groupRolesByAccount`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/lib/roles/resolverRoles.ts#L179-L221): scope-erasing grouping.
* [`ResolverRolesSidebar.tsx::ResolverRolesSidebar`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L146-L515): single `roleName`, "Global roles" label, remove and save handlers.
* [`revokeResolverRoles.ts::prepareRevokeResolverRolesTransaction`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L75): empty name means root, otherwise named scope.
* [`prepareResolverRolesIntent.ts::prepareResolverRolesIntent`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts#L32-L76): passes the pending action through unchanged.
* [`ResolverAddUserSheet.tsx::ResolverAddUserSheet`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L183-L207): root and per-name scopes offered when granting.
* ensjs `revokeResolverRolesWriteParameters` (`@ensdomains/ensjs/wallet/v2`, pinned pkg.pr.new build `e96662c`): `scope: 'root'` encodes `revokeRootRoles`, `scope: 'name'` encodes `authorizeNameRoles`.

## Proof of Concept

## Proof of Concept

A Vitest test inside the Explorer app that uses the real `groupRolesByAccount`, the sidebar's own `roleName` and label expressions copied verbatim, and the real `prepareRevokeResolverRolesTransaction` (which calls the real ensjs encoder). It then decodes the calldata that would be handed to the administrator's wallet. No network, chain or mock of any Portal module is involved; the only stand-in is a minimal wallet client object carrying `account` and `chain`.

Assertions: the three grants collapse into one row labelled "Global roles (all names)"; the selected scope is `alice.eth`; the calldata is `authorizeNameRoles(dns("alice.eth"), ROLE_SET_ADDR | ROLE_SET_TEXT, target, false)`; and it does not start with the `revokeRootRoles` selector.

Audited revision `63772fd872af472ced58b009499355f3430c2a86`; the cited files are unchanged in the audit-comp-ready branch. Node 22, pnpm 10, the repo's existing install and the Explorer `vitest.config.ts`. No environment variables are needed.

Place the file at:

```
apps/portal/test/poc/W15-PORTAL-FLOW-013-MULTIRESOURCE-SCOPE/ProofResolverRoleRemovalLosesScope.test.ts
```

Run from the repository root:

```bash
pnpm install --frozen-lockfile
pnpm --filter portal exec vitest run --config vitest.config.ts test/poc/W15-PORTAL-FLOW-013-MULTIRESOURCE-SCOPE/ProofResolverRoleRemovalLosesScope.test.ts
```

Expected: 1 test file passed, 1 test passed, exit code 0, with the four numbered step logs.

### Proof source

```ts
import {
  permissionedResolverAuthorizeNameRolesSnippet,
  permissionedResolverRevokeRootRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import { describe, expect, it } from 'vitest'
import {
  type Address,
  decodeFunctionData,
  toFunctionSelector,
  toHex,
  type WalletClient,
} from 'viem'
import { packetToBytes } from 'viem/ens'
import { sepolia } from 'viem/chains'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'
import {
  computeNameResource,
  groupRolesByAccount,
  type ResolverRoleKey,
  resolverRoles,
} from '@/lib/roles/resolverRoles'

const ADMIN = '0x1000000000000000000000000000000000000001' as Address
const TARGET = '0x2000000000000000000000000000000000000002' as Address
const RESOLVER = '0x5555555555555555555555555555555555555555' as Address
const ROOT_RESOURCE = `0x${'0'.repeat(64)}`

// Minimal wallet client: the encoder only reads `account` and `chain`.
const walletClient = {
  account: { address: ADMIN, type: 'json-rpc' },
  chain: sepolia,
} as unknown as WalletClient

describe('Explorer "Remove user" loses the resource scope of multi-resource role holders', () => {
  it('encodes a single named-scope authorizeNameRoles instead of revoking the root role', () => {
    // On-chain state as returned by the resolver roles query: the same account
    // holds ROLE_SET_ADDR at the root resource, ROLE_SET_ADDR on alice.eth and
    // ROLE_SET_TEXT on bob.eth. Exactly what ResolverAddUserSheet lets an
    // administrator grant.
    const roles = [
      {
        account: TARGET,
        resource: ROOT_RESOURCE,
        roleBitmap: resolverRoles.ROLE_SET_ADDR.toString(),
      },
      {
        account: TARGET,
        resource: computeNameResource('alice.eth'),
        roleBitmap: resolverRoles.ROLE_SET_ADDR.toString(),
      },
      {
        account: TARGET,
        resource: computeNameResource('bob.eth'),
        roleBitmap: resolverRoles.ROLE_SET_TEXT.toString(),
      },
    ]
    const resourceToName = new Map<string, string>([
      [ROOT_RESOURCE, '(root)'],
      [computeNameResource('alice.eth').toLowerCase(), 'alice.eth'],
      [computeNameResource('bob.eth').toLowerCase(), 'bob.eth'],
    ])

    // 1. Real grouping: one row for the account, scopes collapsed.
    const rows = groupRolesByAccount(roles, resourceToName)
    expect(rows).toHaveLength(1)
    const row = rows[0]!
    expect(row.resolvedNames).toEqual(['(root)', 'alice.eth', 'bob.eth'])
    expect(row.decodedRoles).toEqual([
      'ROLE_SET_ADDR',
      'ROLE_SET_ADDR',
      'ROLE_SET_TEXT',
    ])
    console.log('1. groupRolesByAccount collapsed 3 scopes into one row:', row.resolvedNames)

    // 2. The sidebar's own derivations (ResolverRolesSidebar.tsx lines 166-169
    //    and 393-395), copied verbatim.
    const resolvedNames = row.resolvedNames
    const decodedRoles = row.decodedRoles as ResolverRoleKey[]
    const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
    const scopeLabel = resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`
    expect(scopeLabel).toBe('Global roles (all names)')
    expect(roleName).toBe('alice.eth')
    console.log(`2. sidebar shows "${scopeLabel}" but picks roleName="${roleName}"`)

    // 3. Real transaction builder, called exactly as handleRemoveUser ->
    //    removeUserMutation does: one name, the unioned role list.
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: TARGET,
      roles: decodedRoles,
      walletClient,
      chainId: sepolia.id,
    })
    expect(intent.request.type).toBe('eoa')
    expect(intent.request.to).toBe(RESOLVER)
    expect(intent.request.from).toBe(ADMIN)
    const data = intent.request.data

    // 4. The calldata the administrator's wallet is asked to sign.
    const rootSelector = toFunctionSelector(
      permissionedResolverRevokeRootRolesSnippet[0]!,
    )
    expect(data.startsWith(rootSelector)).toBe(false)

    const decoded = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data,
    })
    expect(decoded.functionName).toBe('authorizeNameRoles')
    const [encodedName, bitmap, account, grant] = decoded.args as [
      `0x${string}`,
      bigint,
      Address,
      boolean,
    ]
    expect(encodedName).toBe(toHex(packetToBytes('alice.eth')))
    expect(bitmap).toBe(resolverRoles.ROLE_SET_ADDR | resolverRoles.ROLE_SET_TEXT)
    expect(account).toBe(TARGET)
    expect(grant).toBe(false)
    console.log(
      `4. calldata = authorizeNameRoles(dns("alice.eth"), 0x${bitmap.toString(16)}, ${account}, false)`,
    )
    console.log(
      '   -> no revokeRootRoles call: the root ROLE_SET_ADDR on every name survives "Remove user",',
    )
    console.log('   -> and bob.eth ROLE_SET_TEXT is never targeted either.')
  })
})
```
