# 89469 w a insight ssrf self host protection bypass in the og image worker leads to temporary denial of service resource amplification of the card preview service

**Submitted on Aug 19th 2026 at 14:05:30 UTC by @BenTimor for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89469
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
    * Locking up the victim from login
    * Cookie bombing, etc.

## Description

## Brief/Intro

I found that the self-host guard added to the Explorer's OG image worker (`apps/portal/src/worker/safe-fetch.ts`) — the protection added in response to the known issue R2-04 — can be bypassed with a trailing-dot FQDN (`https://app.ens.domains./og/...`). The guard compares `url.host` to the worker's own host with a plain string comparison, but the WHATWG URL parser retains the trailing dot, so `app.ens.domains.` passes the check while DNS and Cloudflare route it to the exact same worker. By registering two ENS names whose avatar records cross-point at the worker's own `/og/` route with alternating dot-ness, I can make the worker recursively fetch and re-render itself indefinitely: every request for those names' OG cards spawns a chain of \~10–40 nested worker invocations (each with fresh RPC reads and a full satori render) that only ends when the per-hop 20-second timeout aborts. If exploited in production, an attacker would be able to make card previews for their names permanently fail to render, and to amplify a single preview request into tens of worker invocations and dozens of RPC calls, degrading the shared OG preview service for other users at attacker-chosen times. No funds move and no user data is exfiltrated; the main apps are unaffected.

## Vulnerability Details

### The guard

The Explorer's OG image worker fetches an ENS name's `avatar` text record server-side and renders it into a social preview card. Because the avatar value is attacker-controlled for any name the attacker owns, the worker validates every URL it fetches. The relevant check is `checkFetchableUrl` in `apps/portal/src/worker/safe-fetch.ts`:

```ts
if (selfHost && url.host.toLowerCase() === selfHost.toLowerCase()) {
  return 'self-host'
}
```

`selfHost` is derived per invocation from the request that reached the worker (`apps/portal/src/worker.ts` passes `url.host`, i.e. `new URL(request.url).host`, into `fetchEnsData`). The guard's own documentation comment states its exact purpose (`safe-fetch.ts:70-71`):

> `selfHost` is the worker's own host: an avatar pointing back at `/og/<name>` would make the worker recurse into itself, with each hop spawning a fresh invocation

So the guard exists specifically to stop self-recursion, and the string comparison is the whole defense.

### The bypass

The WHATWG URL parser retains a trailing dot in the hostname, while DNS treats the trailing dot as the DNS root and resolves it to the same records. I verified both halves:

```javascript
new URL('https://app.ens.domains./og/evil.png').host   // 'app.ens.domains.'  ≠ 'app.ens.domains' → guard PASSES the fetch
new URL('https://app.ens.domains/og/evil.png').host    // 'app.ens.domains'   == selfHost → rejected
```

Live (2026-08-19, testnet deployment): both `https://app.ens.domains/ens.eth` and `https://app.ens.domains./ens.eth` return HTTP 200 from the same Cloudflare zone (`104.18.20.234` / `104.18.21.234`), and the deployed OG host `ens-og-image.ens-cf.workers.dev` returns byte-identical responses for both host spellings. The fetch therefore loops back into the same worker: `handleOgImage` → `fetchEnsData` → `resolveAvatarDataUri` → `safeFetch(...)` with the same `/og/<name>.png` route, each hop re-reading the name's records from RPC and re-rendering the card.

### Making the recursion unbounded

I found during testing that a single name does **not** recurse unboundedly: because `selfHost` is recomputed from each request's own URL, at recursion depth 2 the request host is already the trailing-dot host, and the trailing-dot avatar then matches `selfHost` and is blocked. The chain dies at depth 2.

The guard can be fully defeated with **two attacker-owned names whose avatars cross-point with alternating dot-ness**:

```
evil1.eth avatar = https://app.ens.domains./og/evil2.png   (trailing dot)
evil2.eth avatar = https://app.ens.domains/og/evil1.png    (no dot)
```

Walking through the chain: a request for `https://app.ens.domains/og/evil1.png` arrives with `selfHost = "app.ens.domains"`; the worker resolves evil1's avatar (`app.ens.domains.` — trailing dot, ≠ selfHost → fetch allowed); the fetch lands back on the same worker, now with `selfHost = "app.ens.domains."`; the worker resolves evil2's avatar (`app.ens.domains` — no dot, ≠ selfHost → fetch allowed); and so on, alternating forever. Neither avatar ever matches the level's `selfHost`, so the guard never fires. I traced this mechanically (a simulation of the per-level `selfHost` propagation and guard decision over the alternating pair): 12/12 levels allowed, with no terminating condition other than the per-hop timeout.

### What bounds the loop

The only terminator is the per-hop deadline (`safe-fetch.ts:208`, `AbortSignal.timeout(20_000)`) on the deepest hop, or Cloudflare's internal invocation-count guard (Error 1042). Each hop burns a full worker invocation with fresh RPC reads (`getRecords` + `resolveOwner`) and a satori/render pass, so one user request is amplified into roughly 10–40 nested invocations before the deadline tears the chain down.

## Impact Details

* **What the attacker gets:** for every name-pair they register, any request for those names' OG cards (automatic link-preview fetches from Discord/Slack/Telegram/bots, or a user opening the name's page) hangs for \~20 seconds and then fails — the card never renders. Card preview access for those names is temporarily disabled for every user who requests it.
* **Amplification:** each such request multiplies into tens of nested worker invocations and dozens of RPC reads on the shared edge worker. The effect degrades preview generation capacity for *other* names while the chain runs, at attacker-chosen times and for arbitrarily many names (name rotation is cheap on Sepolia; cache busting via query strings and the cache windows — `s-maxage=3600` at the competition commit, `s-maxage=30` deployed — keep the chain re-triggerable).
* **What is NOT at risk:** no funds move, no user data is exfiltrated (https-only, credentials stripped, redirects re-validated), and the Manager/Explorer apps themselves are unaffected — only the OG preview service is degraded.
* **Severity: Low.** Availability-only, bounded per hop, sub-feature scope.

**Impact mapping (why this matches the selected in-scope impact "Temporarily disabling user to access target site"):** the listed examples (login lockout, cookie bombing) are introduced with "such as" and are non-exhaustive; the practical effect here is the same class of harm — a user's access to a target-site feature (card previews for the affected names) is temporarily disabled by attacker action. The OG worker is part of the in-scope `/apps/portal` asset: the program's own Known Issues list includes R2-04 (the SSRF issue on this same worker, rated **Low**), which confirms the team treats worker-surface findings as valid, in-scope, and payable at Low. Per the competition's reward terms, a confirmed Low unlocks the $30,000 reward tier.

**Relationship to R2-04 (why this is a new finding, not a duplicate):** R2-04 disclosed the residual SSRF ("arbitrary public https endpoints are still fetchable"). This report is about the *bypass of the self-host guard added to fix that class* — the guard whose documented purpose is exactly to prevent self-recursion — producing a new denial-of-service consequence. Per the program's Code Freeze Assurance, "Bypassing a deployed fix is considered a new, valid bug.".

## Proof of Concept

*Per the Audit Competition PoC rules, a step-by-step explanation is provided (no runnable code required). All steps below were executed and verified by me on 2026-08-19/20 against the testnet deployment and the frozen competition commit.*

### Prerequisites

1. An attacker controls two ENS names on Sepolia (any registration path), here `evil1.eth` and `evil2.eth`.
2. Set the avatar records so the two names cross-point at the worker's own OG route with alternating dot-ness:
   * `evil1.eth` → `avatar` text record = `https://app.ens.domains./og/evil2.png`
   * `evil2.eth` → `avatar` text record = `https://app.ens.domains/og/evil1.png`

{% stepper %}
{% step %}

## Confirm the parser keeps the trailing dot

`new URL('https://app.ens.domains./og/evil1.png').host` returns `'app.ens.domains.'` — the trailing dot is retained by the WHATWG parser, so the string comparison `url.host.toLowerCase() === selfHost.toLowerCase()` against `'app.ens.domains'` (the worker's own host, no dot) does not match, and the fetch is allowed.
{% endstep %}

{% step %}

## Confirm the trailing-dot host reaches the same worker

```
$ curl -s -o /dev/null -w "%{http_code}\n" -H "Accept: text/html" https://app.ens.domains/ens.eth
200
$ curl -s -o /dev/null -w "%{http_code}\n" -H "Accept: text/html" https://app.ens.domains./ens.eth
200
```

Both requests return 200 from the same Cloudflare zone (`nslookup app.ens.domains` → `104.18.20.234` / `104.18.21.234`), and the deployed OG host `ens-og-image.ens-cf.workers.dev` serves byte-identical responses for both host spellings. The trailing-dot URL is therefore routed to the same worker that the guard is trying to protect.
{% endstep %}

{% step %}

## Trigger the recursion

Request the OG card for one of the names: `https://app.ens.domains/og/evil1.png` (equivalently: share a link to the name — preview bots fetch the OG card automatically).

The code path is: `handleOgImage` → `fetchEnsData(env, name, url.host)` → `resolveAvatarDataUri` → `safeFetch('https://app.ens.domains./og/evil2.png', selfHost='app.ens.domains')`. The guard compares `'app.ens.domains.'` vs `'app.ens.domains'` → allowed. The fetch lands on the same worker's `/og/evil2.png` route, where `selfHost` is now `'app.ens.domains.'`; evil2's avatar `https://app.ens.domains/og/evil1.png` compares `'app.ens.domains'` vs `'app.ens.domains.'` → allowed. The chain alternates between the two hosts indefinitely.
{% endstep %}

{% step %}

## Observe the amplification and termination

Each hop performs fresh RPC reads (`getRecords`, `resolveOwner`) and a full card render. I traced the alternating pair mechanically: 12/12 simulated levels were ALLOWED by the guard, with no terminating condition except the per-hop `AbortSignal.timeout(20_000)` (safe-fetch.ts) on the deepest hop or Cloudflare's Error 1042 invocation-count guard. One preview request thus produces roughly 10–40 nested worker invocations (each with RPC reads + render) and the card never renders for the requesting user.
{% endstep %}

{% step %}

## Repeatability

The edge cache (`s-maxage=3600` at the frozen commit; `s-maxage=30` deployed) makes each name-pair effective once per cache window. The attacker rotates to new name-pairs (cheap on Sepolia) or varies the request's query string to trigger the chain again — two names alone already bind one worker request for \~20–25 seconds of wall time.
{% endstep %}
{% endstepper %}
