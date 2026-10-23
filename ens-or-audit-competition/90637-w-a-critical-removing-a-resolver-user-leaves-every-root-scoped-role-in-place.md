# 90637 w a critical removing a resolver user leaves every root scoped role in place

**Submitted on Aug 26th 2026 at 04:53:25 UTC by @QED for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90637
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer app in this repository (`apps/portal`, called the Portal below) has a page for managing who may write records on an ENSv2 resolver. It offers a "Remove user" button and a dialog promising to "remove this user from all roles". It sends one transaction, addressed to one scope. When the account holds roles at more than one scope only one is cleared, and the one left behind is the resolver's root scope, which applies to every name. The Portal reports success, so nobody retries.

Terms, once. A **resolver** stores a name's records. Each account holds role bits against a **resource**: the **root resource** is `bytes32(0)` and covers *all* names on that resolver, while a **name resource** is `keccak256(namehash(name) ++ bytes32(0))` and covers one name. The app targets Ethereum **Sepolia** only ([`wagmi.ts:67`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/wagmi.ts#L67)).

Commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

## Vulnerability Details

### Root Cause

The roles table groups rows by account for display, merging the bitmaps of every resource into one flat list. The removal handler reuses that display grouping to build a transaction, and picks a scope by taking the first resolved name that is not `(root)`. Root is therefore never picked whenever any name-scoped role also exists. The merged role list goes to that one name scope, and no second transaction is built for root.

Grouping happens in [`resolverRoles.ts:179-221`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L221), which keys only on the account and concatenates `decodedRoles` across resources. The scope is chosen in one line, [`ResolverRolesSidebar.tsx:169`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L169), and reaches the removal at [`:307-322`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L322).

[`revokeResolverRoles.ts:45-61`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L45-L61) turns it into exactly one call: `scope: 'root'` when the name is empty, `scope: 'name'` otherwise. In ensjs those are two different functions, `revokeRootRoles` and `authorizeNameRoles`.

The Portal knows both scopes are involved, because it prints them at [`:391-396`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L396), and still promises full removal at [`:445-448`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L445-L448).

Grouping by account is intended for display, and `resolverRoles.test.ts:187-197` asserts it. The defect is that a destructive action reuses it and then collapses it to one scope.

### Attack Path

{% stepper %}
{% step %}

## Grant a root-scoped role

An operator grants a delegate a root-scoped role, say `ROLE_SET_ADDR`, so the delegate can maintain address records across all names on the resolver.
{% endstep %}

{% step %}

## Grant a name-scoped role

Later the operator also grants that delegate a role scoped to one name, say `ROLE_SET_TEXT` on `sub.eth`. Both scopes are offered by the Portal's own Add-user sheet, whose node picker defaults to "Root (all nodes)" (`ResolverAddUserSheet.tsx:253`).
{% endstep %}

{% step %}

## Remove the delegate

The delegate is no longer trusted. The operator opens the resolver roles page, selects the delegate, clicks "Remove user", and confirms "remove this user from all roles".
{% endstep %}

{% step %}

## Portal revokes only the name-scoped role

The Portal sends one `authorizeNameRoles` call for `sub.eth` with `grant = false`. It sends nothing for the root resource.
{% endstep %}

{% step %}

## Portal reports success

The transaction succeeds and the Portal shows success. The operator stops.
{% endstep %}

{% step %}

## Delegate retains root authority

The delegate still holds `ROLE_SET_ADDR` at root. They can rewrite the address record of every name on that resolver and point it at an address they control.
{% endstep %}
{% endstepper %}

## Impact Details

Revocation is a security control, and here it silently does not run for the scope that matters most. The likely objection is that the delegate retains authority rather than gaining it, so no boundary is crossed. The boundary is the moment the operator decides the delegate is untrusted and acts. The app reports that the decision took effect, and because the dialog says "all roles" the operator does not retry and does not check the chain, so the exposure is indefinite. A retained root `ROLE_SET_ADDR` lets the former delegate change where every name on that resolver resolves, which is direct fund misdirection; a retained root `ROLE_UPGRADE` lets them replace the resolver implementation.

## References

* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169`, `:307-322`, `:391-396`, `:445-448`
* `apps/portal/src/lib/roles/resolverRoles.ts:179-221`
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61`

## Recommended Fix

Build one transaction per resource, not one per account. `groupRolesByAccount` already carries the per-resource `roles` array; use it. For each entry emit `scope: 'root'` when the resource is `bytes32(0)` and `scope: 'name'` otherwise, carrying only that resource's own decoded bits. The transaction modal already supports multi-step flows. Until that lands, the confirmation dialog must not claim "all roles" when `resolvedNames.length > 1`.

## Proof of Concept

The test drives the Portal's own removal path. It builds an indexer result for one delegate holding `ROLE_SET_ADDR` at root and `ROLE_SET_TEXT` at the `sub.eth` resource, runs it through the app's `groupRolesByAccount` and `prepareResolverRolesIntent`, and decodes the calldata the app produced. A control case with root-only roles produces the correct `revokeRootRoles` call, so the test can fail.

Save the fenced diff below as `poc.diff` in the cloned repository root, then run:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
git apply poc.diff
pnpm install --frozen-lockfile
cd apps/portal
npx vitest run src/features/resolver/helpers/poc-5zvy4mxe.test.ts --reporter=verbose
```

```diff
diff --git a/apps/portal/src/features/resolver/helpers/poc-5zvy4mxe.test.ts b/apps/portal/src/features/resolver/helpers/poc-5zvy4mxe.test.ts
new file mode 100644
index 0000000..de3aa28
--- /dev/null
+++ b/apps/portal/src/features/resolver/helpers/poc-5zvy4mxe.test.ts
@@ -0,0 +1,66 @@
+import { createWalletClient, decodeFunctionData, http } from 'viem'
+import { privateKeyToAccount } from 'viem/accounts'
+import { sepolia } from 'viem/chains'
+import { describe, expect, it } from 'vitest'
+import {
+  buildResourceToNameMap,
+  computeNameResource,
+  groupRolesByAccount,
+  resolverRoles as R,
+  type ResolverRoleKey,
+} from '@/lib/roles/resolverRoles'
+import { prepareResolverRolesIntent } from './prepareResolverRolesIntent'
+
+const ROOT = `0x${'00'.repeat(32)}` as const
+const SUB = computeNameResource('sub.eth')
+const ACC = '0x2222222222222222222222222222222222222222' as const
+const ABI = [
+  { type: 'function', name: 'revokeRootRoles',
+    inputs: [{ type: 'uint256' }, { type: 'address' }] },
+  { type: 'function', name: 'authorizeNameRoles',
+    inputs: [{ type: 'bytes' }, { type: 'uint256' }, { type: 'address' }, { type: 'bool' }] },
+] as const
+
+const wallet = createWalletClient({
+  account: privateKeyToAccount(`0x${'11'.repeat(32)}`), chain: sepolia, transport: http(),
+})
+
+/** The portal's "Remove user from all roles" path, over an indexer result. */
+const removeUser = (rows: { resource: string; bits: bigint }[]) => {
+  const [g] = groupRolesByAccount(
+    rows.map((r) => ({ account: ACC, resource: r.resource, roleBitmap: `0x${r.bits.toString(16)}` })),
+    buildResourceToNameMap([{ name: 'sub.eth' }]),
+  )
+  const intent = prepareResolverRolesIntent(
+    { type: 'remove', account: ACC, roles: g.decodedRoles as ResolverRoleKey[],
+      name: g.resolvedNames.find((n) => n !== '(root)') ?? '' }, // ResolverRolesSidebar.tsx:169
+    '0x1111111111111111111111111111111111111111',
+    { walletClient: wallet, chainId: sepolia.id } as never,
+  )
+  if (intent?.request.type !== 'eoa') throw new Error('no intent')
+  const d = decodeFunctionData({ abi: ABI, data: intent.request.data })
+  console.log('resolvedNames :', g.resolvedNames, '| decodedRoles:', g.decodedRoles)
+  console.log('app emits     :', d.functionName, d.args)
+  return { g, d }
+}
+
+describe('5zvy4mxe remove-user', () => {
+  it('CONTROL: root-only roles produce revokeRootRoles', () => {
+    const { d } = removeUser([{ resource: ROOT, bits: R.ROLE_SET_ADDR }])
+    expect(d.functionName).toBe('revokeRootRoles')
+    expect(d.args[0]).toBe(R.ROLE_SET_ADDR)
+  })
+
+  it('BUG: one name-scoped role diverts the revocation away from root', () => {
+    console.log('resources     : root', ROOT, '| sub.eth', SUB)
+    const { g, d } = removeUser([
+      { resource: ROOT, bits: R.ROLE_SET_ADDR },
+      { resource: SUB, bits: R.ROLE_SET_TEXT },
+    ])
+    expect(g.resolvedNames).toEqual(['(root)', 'sub.eth'])
+    expect(d.functionName).toBe('authorizeNameRoles')
+    expect(d.args[3]).toBe(false)
+    expect((d.args[1] as bigint) & R.ROLE_SET_ADDR).toBe(R.ROLE_SET_ADDR)
+    expect(SUB).not.toBe(ROOT)
+  })
+})
```

Output:

```
resolvedNames : [ '(root)' ] | decodedRoles: [ 'ROLE_SET_ADDR' ]
app emits     : revokeRootRoles [ 1n, '0x2222222222222222222222222222222222222222' ]
resources     : root 0x0000000000000000000000000000000000000000000000000000000000000000 | sub.eth 0x30b25859ea158c987f67622745681704cd6f7cb1dc48b8ecbb22081fdc864151
resolvedNames : [ '(root)', 'sub.eth' ] | decodedRoles: [ 'ROLE_SET_ADDR', 'ROLE_SET_TEXT' ]
app emits     : authorizeNameRoles [
  '0x037375620365746800',
  17n,
  '0x2222222222222222222222222222222222222222',
  false
]

 ✓ CONTROL: root-only roles produce revokeRootRoles 2ms
 ✓ BUG: one name-scoped role diverts the revocation away from root 1ms

 Test Files  1 passed (1)
      Tests  2 passed (2)
```

`17n` is `ROLE_SET_ADDR | ROLE_SET_TEXT` and `0x037375620365746800` is DNS-encoded `sub.eth`. The one call the app emits targets the `sub.eth` resource. `ROLE_SET_ADDR` lives at the root resource, which that call never touches, so it survives.
