# 90636 w a insight unverified default reverse name is displayed as a verified primary name

**Submitted on Aug 26th 2026 at 04:52:04 UTC by @QED for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90636
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer app in this repository (`apps/portal`, called the Portal below) has an address page that lists, per network, the primary name of an address. For L2 networks with no reverse record of their own, the Portal falls back to the L1 `default.reverse` name. It fetches that name with forward verification switched off, throws the verification result away, and then hard-codes those rows to render "Forward match: True" and a "Primary name" badge. The row's detail panel shows a green banner reading "This is the primary name on ".

Anyone can write `default.reverse` for their own address, with no permission and no check that they own the name they write. So any address can make the Portal state, with a green tick, that it is `vitalik.eth` on Optimism, Base, Arbitrum, Linea and Scroll. This is not a metadata display gap; it is a positive identity assertion produced by a query that deliberately disabled the check that would have refuted it.

Terms, defined once. **Reverse resolution** maps an address to a name; it is only trustworthy when the name's forward record resolves back to that address (the "forward match"). ENSIP-19 defines `default.reverse`, coin type `0x80000000`, as a fallback primary name for any EVM chain with no chain-specific reverse record. The app targets Ethereum **Sepolia** and the Sepolia testnets of the listed L2s ([`wagmi.ts:67`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/wagmi.ts#L67)).

Commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

## Vulnerability Details

### Root Cause

The `default.reverse` lookup passes `allowMismatch: true`. In ensjs that flag changes the meaning of a failed forward check from "return nothing" to "return the name and set `match: false`". The Portal keeps only `.name` and discards `.match`. Downstream, the display state treats inheritance itself as proof: `isPrimaryName` is true whenever a row inherits the default, without consulting the forward match at all. The table cell for the inherited case does not even read `forwardMatch`; it returns a fixed "True" badge and a "Primary name" badge.

The fetch is at [`useReverseResolution.ts:236-241`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts#L236-L241); line 241 is `const defaultName = defaultResult?.name ?? null`, which is where `match` is dropped. The ensjs behaviour is at `src/actions/public/getName.ts:138-148` of the pinned `@ensdomains/ensjs@5.0.0-alpha.1`: on a `ReverseAddressMismatch` revert it sets `match = false` when `allowMismatch` is given and returns `null` when it is not. The display state is at [`computeDisplayNameState.ts:77-88`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts#L77-L88), where `isPrimaryName = forwardMatch || isInheritingDefault`. The hard-coded cell is at [`columns.tsx:91-111`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx#L91-L111), and `canInheritDefault` at [`:12-13`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx#L12-L13) covers every row that is not mainnet and not the Default row itself, which is every L2.

The app has an explicit "unverified" state and this path never reaches it. The detail panel renders a green banner at [`ReverseResolutionSidebar.tsx:645-652`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx#L645-L652) and the red alternative at [`:654-661`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx#L654-L661), selected by the same `isPrimaryName` that inheritance forces true.

### Attack Path

{% stepper %}
{% step %}

## Choose an address and name

The attacker controls address `A` and picks a well-known name they do not own, `victim.eth`.
{% endstep %}

{% step %}

## Set the default reverse name

From `A`, the attacker calls `setName("victim.eth")` on the Sepolia `DefaultReverseRegistrar` (`0x4f382928805ba0e23b30cfb75fc9e848e82dfd47`, named in [`config.ts:34-35`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/config.ts#L34-L35)).

The registrar does not check that `A` owns `victim.eth`, and `victim.eth`'s forward records do not point at `A`.
{% endstep %}

{% step %}

## Allow L2 inheritance

The attacker sets no chain-specific L2 reverse record, so every L2 row falls through to inheritance.
{% endstep %}

{% step %}

## Direct users to the address page

The attacker sends the victim a link to `/addr/A/reverse-resolution`, or simply relies on anyone looking `A` up in the Portal.
{% endstep %}

{% step %}

## Portal displays a verified-looking identity

Each L2 row renders `victim.eth`, a "True" forward-match badge and a "Primary name" badge. Opening the row shows the green "This is the primary name on " banner.
{% endstep %}

{% step %}

## Victim sends funds

The reader concludes `A` is the address behind `victim.eth` on those chains and pays it.
{% endstep %}
{% endstepper %}

## Impact Details

The app displays an incorrect name for an address, with an affirmative verification badge, at attacker choice and at no cost beyond one Sepolia transaction. The Portal is the surface a user consults precisely when they want to check who an address is. Its answer here is the attacker's claim, presented as verified.

The obvious defence — the reader checking the name themselves — is exactly what the green banner and the "True" badge are there to replace. The Portal has a correct, red, "does not resolve back" state for the identical underlying condition on other rows; a reader who has seen that state has every reason to trust its absence.

There is no precondition on the victim. `victim.eth` can be any existing name, held by anyone, with no interaction from its owner.

## References

* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-241`
* `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:77-88`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:12-13`, `:91-111`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx:645-652`, `:654-661`
* `@ensdomains/ensjs@5.0.0-alpha.1` `src/actions/public/getName.ts:138-148`

## Recommended Fix

Carry the verification result instead of discarding it. Keep `defaultResult.match` alongside `defaultName` on every row that inherits it, make `isPrimaryName` require it (`forwardMatch || (isInheritingDefault && defaultMatch)`), and make the inherited branch of the forward-match column render from that value rather than from a fixed "True". An inherited name whose forward check failed should get the red "does not resolve back to this name" treatment the app already implements, not the green one.

## Proof of Concept

The test below drives ensjs exactly as the app calls it, against a simulated Sepolia UniversalResolver that returns the same `ReverseAddressMismatch` revert a real one returns when the forward record does not match. It shows the mismatch is detected and then discarded, feeds the result through the app's own `computeDisplayNameState`, and renders the app's real table columns. The mainnet row is rendered from the identical state as a control and correctly reads "False".

Save the fenced diff below as `poc.diff` in the cloned repository root, then run:

```bash
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
git apply poc.diff
pnpm install --frozen-lockfile
cd apps/portal
npx vitest run src/features/reverse-resolution/poc-bjsm01wy.test.tsx --reporter=verbose
```

```diff
diff --git a/apps/portal/src/features/reverse-resolution/poc-bjsm01wy.test.tsx b/apps/portal/src/features/reverse-resolution/poc-bjsm01wy.test.tsx
new file mode 100644
index 0000000..f7bc7ed
--- /dev/null
+++ b/apps/portal/src/features/reverse-resolution/poc-bjsm01wy.test.tsx
@@ -0,0 +1,96 @@
+import { universalResolverReverseSnippet } from '@ensdomains/ensjs-abi/universalResolver'
+import { getName } from '@ensdomains/ensjs/public'
+import { flexRender, getCoreRowModel, useReactTable } from '@tanstack/react-table'
+import { render, screen, within } from '@testing-library/react'
+import { createPublicClient, custom, encodeErrorResult, RpcRequestError } from 'viem'
+import { describe, expect, it } from 'vitest'
+import { DEFAULT_EVM_COIN_TYPE, MAINNET_COIN_TYPE } from '@/lib/coinType'
+import { sepoliaWithEns } from '@/test-utils/wagmi.mock'
+import { computeDisplayNameState } from '@/utils/reverseResolution/computeDisplayNameState'
+import { columns } from './components/ReverseResolutionTable/columns'
+import { REVERSE_RESOLUTION_NETWORKS } from './config'
+import type { ReverseResolutionResult } from './hooks/useReverseResolution'
+
+const ATTACKER = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8' as const
+const NAME = 'victim.eth'
+
+/** A Sepolia node holding `victim.eth` as the attacker's `default.reverse`
+ *  record, rejecting the forward check as it does for any name they do not own. */
+const client = createPublicClient({
+  chain: sepoliaWithEns,
+  transport: custom({
+    async request({ method }: { method: string }) {
+      if (method === 'eth_chainId') return '0xaa36a7'
+      throw new RpcRequestError({ body: {}, url: '', error: { code: 3, message: 'execution reverted',
+        data: encodeErrorResult({ abi: universalResolverReverseSnippet,
+          errorName: 'ReverseAddressMismatch', args: [NAME, ATTACKER] }) } })
+    },
+  }),
+})
+
+const Rows = ({ data }: { data: ReverseResolutionResult[] }) => {
+  const table = useReactTable({ data, columns, getCoreRowModel: getCoreRowModel() })
+  return (
+    <table><tbody>
+      {table.getRowModel().rows.map((r) => (
+        <tr key={r.id} data-testid={`row-${r.original.label}`}>
+          {r.getVisibleCells().map((c) => (
+            <td key={c.id}>{flexRender(c.column.columnDef.cell, c.getContext())}</td>
+          ))}
+        </tr>
+      ))}
+    </tbody></table>
+  )
+}
+
+describe('bjsm01wy — an unverified default.reverse name is shown as verified', () => {
+  it('the forward check fails; allowMismatch is what keeps the name alive', async () => {
+    const strict = await getName(client, { address: ATTACKER, coinType: DEFAULT_EVM_COIN_TYPE })
+    const asPortalCallsIt = await getName(client, { address: ATTACKER,
+      coinType: DEFAULT_EVM_COIN_TYPE, allowMismatch: true }) // useReverseResolution.ts:236-240
+    console.log('[getName] without allowMismatch:', strict)
+    console.log('[getName] as the portal calls it:', asPortalCallsIt)
+    expect(strict).toBeNull()
+    expect(asPortalCallsIt).toMatchObject({ name: NAME, match: false })
+  })
+
+  it('BUG: every L2 row renders "True" and "Primary name" for that name', async () => {
+    const result = await getName(client, { address: ATTACKER,
+      coinType: DEFAULT_EVM_COIN_TYPE, allowMismatch: true })
+    const defaultName = result?.name ?? null // :241 keeps only `.name`; `.match` is dropped
+
+    // The row shape getReverseResolution produces for an address with no
+    // reverse record of its own: createEmptyResult(network) plus defaultName.
+    const emptyRow = (n: (typeof REVERSE_RESOLUTION_NETWORKS)[number]) => ({
+      ...n, name: null, reverseResolverAddress: null, resolverAddress: null,
+      normalized: true, forwardMatch: false, defaultName,
+    }) satisfies ReverseResolutionResult
+    const l2 = emptyRow(REVERSE_RESOLUTION_NETWORKS.find((n) =>
+      n.coinType !== DEFAULT_EVM_COIN_TYPE && n.coinType !== MAINNET_COIN_TYPE)!)
+    const mainnet = emptyRow(REVERSE_RESOLUTION_NETWORKS.find((n) => n.coinType === MAINNET_COIN_TYPE)!)
+
+    const state = computeDisplayNameState({ name: l2.name, defaultName: l2.defaultName,
+      forwardMatch: l2.forwardMatch, reverseRegistrarChainId: l2.reverseRegistrarChainId! })
+    console.log('[state] row            :', l2.label)
+    console.log('[state] forwardMatch in:', l2.forwardMatch)
+    console.log('[state] computed       :', state)
+    expect(l2.forwardMatch).toBe(false)
+    expect(state.isPrimaryName).toBe(true)
+    expect(state.displayName).toBe(NAME)
+
+    render(<Rows data={[l2, mainnet]} />)
+    const l2Row = within(screen.getByTestId(`row-${l2.label}`))
+    const mainnetRow = within(screen.getByTestId(`row-${mainnet.label}`))
+    console.log(`[render] ${l2.label} row:`, screen.getByTestId(`row-${l2.label}`).textContent)
+    console.log(`[render] ${mainnet.label} row :`, screen.getByTestId(`row-${mainnet.label}`).textContent)
+
+    expect(l2Row.getByText(NAME)).toBeInTheDocument()
+    expect(l2Row.getByText('True')).toBeInTheDocument()
+    expect(l2Row.getByText('Primary name')).toBeInTheDocument()
+    expect(l2Row.queryByText('False')).toBeNull()
+
+    // CONTROL: the mainnet row cannot inherit, and reports the honest verdict.
+    expect(mainnetRow.getByText('False')).toBeInTheDocument()
+    expect(mainnetRow.queryByText('Primary name')).toBeNull()
+  })
+})
```

Output:

```
stdout | poc-bjsm01wy.test.tsx > the forward check fails; allowMismatch is what keeps the name alive
[getName] without allowMismatch: null
[getName] as the portal calls it: {
  name: 'victim.eth',
  match: false,
  normalized: true,
  reverseResolverAddress: null,
  resolverAddress: null
}
stdout | poc-bjsm01wy.test.tsx > BUG: every L2 row renders "True" and "Primary name" for that name
[state] row            : Optimism
[state] forwardMatch in: false
[state] computed       : {
  displayName: 'victim.eth',
  isInheritingDefault: true,
  isPrimaryName: true,
  canSetAsPrimary: false
}
[render] Optimism row: Optimismvictim.ethDefaultTruePrimary nameDefault
[render] Mainnet row : MainnetnullFalse

 ✓ the forward check fails; allowMismatch is what keeps the name alive 85ms
 ✓ BUG: every L2 row renders "True" and "Primary name" for that name 19ms

 Test Files  1 passed (1)
      Tests  2 passed (2)
```

`match: false` is the app's own dependency reporting that the forward check failed. The rendered Optimism row says `victim.eth`, `True`, `Primary name`. `isPrimaryName: true` is what selects the green banner in the detail panel.
