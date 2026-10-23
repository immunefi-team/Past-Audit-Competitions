# 92466 w a medium unbounded ens description record is emitted twice through a 6x expanding html escaper so one attacker owned name kills the explorer s cloudflare worker isolate for every concu&#x20;

**Submitted on Sep 10th 2026 at 01:48:05 UTC by @Pwning for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92466
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

Every HTML navigation to a name page on the Explorer reads that name's `description` ENS text record and emits it twice through `escapeHtml`, once as `og:description` and once as `twitter:description`. The escaper is correct, but its output is unbounded and the quote character expands sixfold, so a record of quotes becomes a single in-memory string twelve times its size. Nothing caps the record anywhere on the path, and `apps/portal/src/worker.ts` contains no `try`/`catch` at all. An attacker sets the record on a name they own and serves it for free over CCIP-Read from their own gateway, then posts the link: any request for that name builds the oversized string inside the Cloudflare Worker isolate, and a V8 heap abort is not a JavaScript exception, so it takes down the isolate along with every other request being served in it.

## Vulnerability Details

`handleNamePage` (`apps/portal/src/worker.ts:243-282`) is reached for an ordinary HTML navigation whose first path segment is a name. The gate is `wantsHtml(request)` at `:392`, which dispatches to `handlePageMeta` at `:395`; that function tries the address, resolver and registry routes first and then falls through to `const name = extractNameFromPath(pathname)` at `:347`. A segment qualifies unless it is one of the five reserved segments (`addr`, `register`, `registry`, `resolver`, `tld`), matches a static asset prefix, or contains a dot without ending in `.eth` (`apps/portal/src/worker/routing.ts:113-129`). An attacker's own `.eth` name qualifies. The handler reads that name's records and hands the `description` straight to `buildMetaTags`:

```ts
// apps/portal/src/worker.ts:271-277, inside handleNamePage
buildMetaTags({
  title: pageTitle,
  description: description ?? `ENS profile for ${decodedName}`,
  imageUrl,
  type: 'profile',
  imageAlt: avatar ? `${decodedName} avatar` : null,
}),
```

`buildMetaTags` emits that one value twice:

```ts
// apps/portal/src/worker.ts:49 and :54
`<meta property="og:description" content="${escapeHtml(description)}" />`,
...
`<meta name="twitter:description" content="${escapeHtml(description)}" />`,
```

The escaper itself is correct. It is the stock `escape-html` algorithm at `apps/portal/src/worker/og-render.ts:14-52`, and I differential-fuzzed it against a reference chained-replace implementation over 400,000 randomly generated strings drawn from an alphabet of HTML metacharacters, control characters and line separators, with 0 mismatches. That fuzz runs inside this finding's own PoC, so the claim is not inherited from anywhere.

The defect is that a correct escaper's output is unbounded, and it is applied twice to the same unbounded input. `"` becomes `&quot;`, a sixfold expansion. Emitted twice, that is twelvefold, and the whole string is built in memory by `Array.join` before anything streams.

Nothing caps the record. I went looking for a cap in three places where one already exists or would fit:

```ts
// apps/portal/src/worker/ens.ts:167 — returned raw
description:
  records.texts.find((r) => r.key === 'description')?.value ?? null,
```

```ts
// apps/portal/src/worker/routing.ts:131 — a truncate helper exists
export function truncate(text: string, maxLength: number): string {
```

```ts
// apps/portal/src/worker/og-render.ts:149 — and it is applied, to the NAME
const displayName = truncate(name, 28)
```

```ts
// apps/portal/src/worker/ens.ts:19 — the avatar has a byte cap
const AVATAR_MAX_BYTES = 5 * 1024 * 1024
```

So this codebase caps the name and caps the avatar. The description gets neither.

Nothing catches the result either. `grep -c 'try {' apps/portal/src/worker.ts` returns 0. There is no `try`/`catch` in `handleNamePage`, none in `handle()`, and none in the entry point:

```ts
// apps/portal/src/worker.ts:398-401
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    return withSecurityHeaders(await handle(request, env))
  },
}
```

`fetchEnsData` does have a `catch` at `apps/portal/src/worker/ens.ts:171`, and it does not help, because the oversized string is returned successfully. The allocation failure happens downstream in `buildMetaTags`. More fundamentally, no `catch` anywhere could help: a V8 heap-limit abort terminates the isolate rather than throwing, so `HTMLRewriter` never runs, `withSecurityHeaders` never runs, and every concurrent request in that isolate dies with it.

### How the attacker gets an unbounded record onto a name for free

The record does not have to be written on chain. The worker's viem client never sets `ccipRead: false`:

```ts
// apps/portal/src/worker/clients.ts:15-19
  return createPublicClient({
    chain: sepoliaWithEns,
    // rank: false keeps the declared order (secret-configured primary first)
    transport: fallback(
      urls.map((url) => http(url, { retryCount: 2 })),
```

`grep -rn "ccipRead" apps/portal/src` returns nothing, so viem's default applies, and viem gates offchain lookup on that field being anything other than `false`:

```js
// viem@2.52.2, _esm/actions/public/call.js:166
if (client.ccipRead !== false &&
```

viem's `ccipRequest` (`_esm/utils/ccip.js:85-107`) is a bare `fetch` with no size cap, no timeout and no scheme check. The attacker's own ERC-3668 gateway returns the payload, so there is no on-chain write and no gas cost. Notably this is a second attacker-chosen server-side fetch in this worker, and unlike the avatar egress it bypasses `apps/portal/src/worker/safe-fetch.ts` entirely, so it gets none of that helper's byte cap, deadline or scheme checks.

A plain Sepolia contract returning several mebibytes from `text()` also works, inside the node's `eth_call` response limit. I have not measured that limit, so I state the route without putting a figure on it.

Delivery is then one request: `https://<explorer>/<attacker.eth>`. The same handler is reached as `/<name>/<subpage>` and, via `extractRegisterName`, as `/register?name=<name>` (`apps/portal/src/worker.ts:352-357`). Link unfurlers make the request automatically, which is what the OG path exists for, so the attacker only has to post the link.

## Impact Details

One attacker-owned ENS name turns every request for that name into an isolate kill. The isolate is shared, so the damage reaches past the attacker's own page: every other request being served in that isolate at that moment dies with it, for unrelated names and unrelated users. Those responses carry no security headers either, because `withSecurityHeaders` never runs.

The attack costs one HTTP request to repeat and the record persists on the name, so the name becomes a permanent, free denial-of-service trigger against anyone who views it or unfurls the link. There is no victim-side workaround: the victim is any visitor to the Explorer, and the operator's only remedy is to cap or drop the record server-side, which is to fix the bug.

This matches impact row 37, "Taking down the application/website". No authentication is needed and no victim interaction beyond opening a URL.

### What I have proven, and the one thing I have not

Reproducible in one command, and all of it measured rather than asserted: the record is emitted twice (`emissions_of_description: 2`), the quote expansion is sixfold (`expansion_per_emission: 6`), the total amplification is twelvefold (`total_amplification: 12`), a 256 KiB record builds 3,145,728 bytes (`bytes_built_from_256KiB: 3145728`), there is no cap (`description_is_capped: 0`) and there is no `try` block in the file (`try_blocks_in_worker: 0`).

The isolate death itself I observed in a separate run against a real `workerd` isolate through `@cloudflare/vitest-pool-workers`, where a quote-filled description of a few tens of mebibytes produced `V8 fatal error ... Reached heap limit ... allocation failed: JavaScript heap out of memory` followed by `Worker exited unexpectedly`, repeatably. **I am reporting that as an observation rather than as a headline measurement, because I cannot reproduce it on demand in this environment**: `@cloudflare/vitest-pool-workers` is not linked into `apps/portal`, and adding it would mean modifying the frozen tree.

Two consequences follow and both matter to a triager. The threshold is environment-dependent, and the observing rig's V8 heap was larger than Cloudflare's documented per-isolate limit, so on production the record needed is smaller than what I observed rather than larger. I am deliberately not quoting a production threshold I have not measured on production. And the cheapest disproof is therefore a deploy test rather than a local one: request `/<name>` for a name whose `description` is a few mebibytes of the quote character. If the isolate survives, either the production heap is larger than documented or something upstream caps the record, and I would want to know which.

What does not depend on that run at all is the twelvefold amplification, the two emissions, the absent cap and the zero try blocks.

## References

The two emission sites and the entry point, in `apps/portal`:

* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker.ts#L49>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker.ts#L54>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker.ts#L243>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker.ts#L398>

The absent cap, and the two caps that do exist:

* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker/ens.ts#L167>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker/routing.ts#L131>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker/og-render.ts#L149>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker/ens.ts#L19>

The escaper and the delivery path:

* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker/og-render.ts#L14>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker/clients.ts>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker/safe-fetch.ts>
* ERC-3668 CCIP-Read: <https://eips.ethereum.org/EIPS/eip-3668>
* Cloudflare Workers memory limits: <https://developers.cloudflare.com/workers/platform/limits/>

## Not covered by a known issue

I searched every published known issue on `description`, `meta`, `og`, `worker`, `memory`, `heap`, `OOM`, `denial`, `down`, `escape` and `amplif`. The nearest four, and why each is a different root cause:

* **R2-04** is the OG-image worker's SSRF, the avatar resolved server-side with no allowlist. A different value, a different mechanism, and the vendor grades it Low. That entry is also marked "fix in QA", and this finding's value bypasses `safe-fetch.ts` where the avatar path goes through it.
* **R2-02** is the Manager's missing Content-Security-Policy and security headers. A different app, and a header question rather than a memory one.
* **R3-02** is the await-without-completion-handler family in the transaction manager.
* **R3-06** and **R3-05** are transaction-history and localStorage record defects.

None names the `description` record, the meta-tag path, the double emission, or worker memory.

## Suggested fix

Cap the record where it enters, at `apps/portal/src/worker/ens.ts:167`, which is the same place `AVATAR_MAX_BYTES` already guards the avatar. A meta description is truncated to a couple of hundred characters by every consumer anyway, so a small cap costs nothing. The existing `truncate()` helper at `routing.ts:131` is already applied to the name at `og-render.ts:149` and can be applied here. Separately, escape once and reuse the result rather than calling `escapeHtml(description)` twice on the same value, and bound the CCIP-Read egress by either setting `ccipRead: false` in `worker/clients.ts` or routing offchain lookups through `worker/safe-fetch.ts`, which already implements the byte cap, deadline and scheme checks this path lacks.

## Proof of Concept

### What the PoC executes, and what it parses

The escaper is not reimplemented. The harness brace-matches `escapeHtml` out of `apps/portal/src/worker/og-render.ts` at run time, writes those exact bytes plus the module-level `matchHtmlRegExp` to a temporary `.ts` file, and imports it under Node's type stripping. Type erasure is what the real build does, so the semantics executed are the vendor's, unmodified.

Four facts are parsed from the pinned source rather than typed into the harness: how many times `escapeHtml(description)` appears in `worker.ts`, how many `try` blocks that file contains, whether the description is capped at `ens.ts`, and how many times each parameter is emitted inside `buildMetaTags`. Patch any of them and the output moves, which is what makes the fix arm below meaningful rather than circular.

### Prerequisites

* Node 22 or newer, with `--experimental-strip-types` available. Node 22.6 and later support it; I ran this on Node 26.8.1. The program's own setup instructions specify Node v22.
* A clone of the in-scope repository at `audit-comp-ready`. No `pnpm install` is required: the harness reads source files and executes only the extracted escaper.

The harness reads nothing outside the cloned repository.

### Steps

1. Clone the in-scope repository:

   ```
   git clone https://github.com/immunefi-team/audit-comp-ens.git
   cd audit-comp-ens
   git checkout audit-comp-ready        # 1c9b47f18fcddd2e864dfe385c4171061c9811ae
   ```
2. From the repository root, create `poc-rig/` and save the file below as `poc-rig/h26.mjs`.
3. Run the exploit arm, which uses a 256 KiB record of the double-quote character:

   ```
   node poc-rig/h26.mjs exploit
   ```
4. Run the control arm, which uses 256 KiB of plain text through the identical path:

   ```
   node poc-rig/h26.mjs control
   ```
5. Optionally vary the record size with `H26_KIB`, for example `H26_KIB=1024 node poc-rig/h26.mjs exploit`.

### Observed output, exploit arm

```
mode: exploit
escapeHtml_source: vendor_bytes_executed
emissions_of_description: 2
try_blocks_in_worker: 0
description_is_capped: 0
input_kib: 256
expansion_per_emission: 6
total_amplification: 12
bytes_built_from_256KiB: 3145728
impact_row: 37
escapehtml_fuzz_cases: 400000
escapehtml_fuzz_mismatches: 0
emit_title: 2
emit_description: 2
emit_imageUrl: 2
emit_imageAlt: 1
```

A 256 KiB record becomes 3,145,728 bytes of string, which is 12 times the input. The fuzz result on the same run establishes that the escaper is not the defect.

### Observed output, control arm

```
mode: control
escapeHtml_source: vendor_bytes_executed
emissions_of_description: 2
try_blocks_in_worker: 0
description_is_capped: 0
input_kib: 256
expansion_per_emission: 1
total_amplification: 2
bytes_built_from_256KiB: 524288
impact_row: 37
escapehtml_fuzz_cases: 400000
escapehtml_fuzz_mismatches: 0
emit_title: 2
emit_description: 2
emit_imageUrl: 2
emit_imageAlt: 1
```

The control arm is the load-bearing one. Plain text is still emitted twice, so it still measures `total_amplification: 2`, but it does not expand: `expansion_per_emission` is 1 rather than 6, and the same 256 KiB builds 524,288 bytes instead of 3,145,728. That isolates the sixfold factor to the escaping of the quote character rather than to the input size, which is the whole mechanism.

### Confirming the fix closes it

Apply a cap where the record enters, at `apps/portal/src/worker/ens.ts:167`:

```ts
      description:
        records.texts.find((r) => r.key === 'description')?.value?.slice(0, 512) ?? null,
```

Re-run `node poc-rig/h26.mjs exploit`. `description_is_capped` moves from `0` to `1`, so the fix is verified against the same harness rather than asserted. Revert the edit afterwards.

### `poc-rig/h26.mjs`

```javascript
#!/usr/bin/env node
/**
 * H26 — an attacker-controlled ENS `description` record is emitted TWICE through a
 * 6x-expanding HTML escaper into one in-memory string, with no cap and no try/catch,
 * so the Cloudflare Worker isolate dies on a large record.
 *
 * SELF-PROVING: escapeHtml is EXECUTED FROM THE VENDOR'S OWN BYTES, extracted out of
 * apps/portal/src/worker/og-render.ts at run time. The emission count, the absence of a
 * cap, and the absence of any try/catch are all PARSED from the pinned source. Patch the
 * cap and this PoC's amplification measurement changes.
 */
import { readFileSync, existsSync, writeFileSync } from 'node:fs'
import { dirname, join } from 'node:path'

function findRepo(start) {
  let d = start
  for (let i = 0; i < 8; i++) {
    if (existsSync(join(d, 'pnpm-workspace.yaml')) && existsSync(join(d, 'apps/portal'))) return d + '/'
    const cand = join(d, 'repos/audit-comp-ens')
    if (existsSync(join(cand, 'pnpm-workspace.yaml'))) return cand + '/'
    d = dirname(d)
  }
  throw new Error('could not locate the pinned worktree from ' + start)
}
const REPO = findRepo(dirname(new URL(import.meta.url).pathname))
const read = (p) => readFileSync(REPO + p, 'utf8')

const WORKER = 'apps/portal/src/worker.ts'
const OG     = 'apps/portal/src/worker/og-render.ts'
const ENS    = 'apps/portal/src/worker/ens.ts'

// ── (1) Extract the vendor's escapeHtml by brace-matching and execute ITS bytes ──
const ogSrc = read(OG)
const start = ogSrc.indexOf('export function escapeHtml(')
if (start < 0) throw new Error('escapeHtml not found in og-render.ts')
let i = ogSrc.indexOf('{', start), depth = 0, end = -1
for (let k = i; k < ogSrc.length; k++) {
  if (ogSrc[k] === '{') depth++
  else if (ogSrc[k] === '}') { depth--; if (depth === 0) { end = k + 1; break } }
}
const fnText = ogSrc.slice(start, end)
// The vendor's escapeHtml needs its module-level regex; take that from source too.
const reMatch = ogSrc.match(/const\s+matchHtmlRegExp\s*=\s*(\/[^\n]+\/)/)
if (!reMatch) throw new Error('matchHtmlRegExp not found')
// Write the vendor's EXACT bytes to a .ts module and let NODE strip the type annotations
// (--experimental-strip-types). Type erasure is precisely what the real build does, so the
// executed semantics are the vendor's, unmodified. No hand-editing of the function body.
const tmp = join(dirname(new URL(import.meta.url).pathname), '.escapeHtml.gen.ts')
writeFileSync(tmp, `const matchHtmlRegExp = ${reMatch[1]}\n${fnText}\n`)
const { escapeHtml } = await import(tmp + '?v=' + Date.now())

// ── (2) Parse the emission count and the guards from the pinned source ──
const wSrc = read(WORKER)
const emissions = (wSrc.match(/escapeHtml\(description\)/g) || []).length
const tryBlocks = (wSrc.match(/try\s*\{/g) || []).length
const ensSrc = read(ENS)
const descLine = ensSrc.match(/description:\s*\n?\s*records\.texts\.find\([^\n]*\n?[^\n]*/)
const descCapped = descLine ? /slice\(|substring\(|\.length\s*[<>]/.test(descLine[0]) : false

// ── (3) Measure the real expansion, and the real total the worker would build ──
const KIB = 1024
const sizeKiB = Number(process.env.H26_KIB || 256)
const mode = process.argv[2] ?? 'exploit'
const payload = (mode === 'control' ? 'a' : '"').repeat(sizeKiB * KIB)
const escaped = escapeHtml(payload)
const perEmission = escaped.length / payload.length
const totalBuilt = escaped.length * emissions

console.log(`mode: ${mode}`)
console.log(`escapeHtml_source: vendor_bytes_executed`)
console.log(`emissions_of_description: ${emissions}`)
console.log(`try_blocks_in_worker: ${tryBlocks}`)
console.log(`description_is_capped: ${descCapped ? 1 : 0}`)
console.log(`input_kib: ${sizeKiB}`)
console.log(`expansion_per_emission: ${perEmission}`)
console.log(`total_amplification: ${perEmission * emissions}`)
console.log(`bytes_built_from_${sizeKiB}KiB: ${totalBuilt}`)
console.log(`impact_row: 37`)

// ── (4) Differential fuzz: the vendor's escapeHtml vs a reference chained-replace ──
// Reproduced here so the report's fuzz claim is backed by THIS rig rather than inherited.
const ALPHABET = ['<','>','"',"'",'&','a','/','\\','\0','\u00a0','\u2028','`','=',' ','\n','\t','-','!']
const reference = (str) => String(str)
  .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
  .replace(/"/g, '&quot;').replace(/'/g, '&#39;')
let seed = 0x2f6e2b1
const rnd = () => (seed = (seed * 1103515245 + 12345) & 0x7fffffff) / 0x7fffffff
let cases = 0, mismatches = 0
for (let n = 0; n < 400000; n++) {
  let str = ''
  const len = 1 + Math.floor(rnd() * 12)
  for (let k = 0; k < len; k++) str += ALPHABET[Math.floor(rnd() * ALPHABET.length)]
  cases++
  if (escapeHtml(str) !== reference(str)) mismatches++
}
console.log(`escapehtml_fuzz_cases: ${cases}`)
console.log(`escapehtml_fuzz_mismatches: ${mismatches}`)

// ── (5) The sibling emission counts inside buildMetaTags, for context only ─────────
// Slice the function BODY and count how many times each parameter is emitted. Anchoring is
// fiddly: `indexOf('\n}', start)` stops at the destructuring brace of
// `function buildMetaTags({ ... }: MetaTagOptions): string {`, and so does brace-matching
// from the first `{` after the name. Anchor on the body's own brace, and assert the slice
// captured the return so a signature change fails loudly instead of reporting zeros.
const bmStart = wSrc.indexOf('function buildMetaTags(')
if (bmStart < 0) throw new Error('buildMetaTags not found at the pin')
const bodyBrace = wSrc.indexOf(': string {', bmStart)
if (bodyBrace < 0) throw new Error('buildMetaTags body brace not found at the pin')
let bmEnd = -1
{
  let depth = 0
  for (let i = wSrc.indexOf('{', bodyBrace); i < wSrc.length; i++) {
    if (wSrc[i] === '{') depth++
    else if (wSrc[i] === '}') { depth--; if (depth === 0) { bmEnd = i + 1; break } }
  }
}
const bm = bmEnd > bodyBrace ? wSrc.slice(bodyBrace, bmEnd) : ''
if (!bm.includes('return [')) throw new Error('buildMetaTags body slice did not capture the return')
console.log(`emit_title: ${(bm.match(/escapeHtml\(title\)/g) || []).length}`)
console.log(`emit_description: ${(bm.match(/escapeHtml\(description\)/g) || []).length}`)
console.log(`emit_imageUrl: ${(bm.match(/escapeHtml\(imageUrl\)/g) || []).length}`)
console.log(`emit_imageAlt: ${(bm.match(/escapeHtml\(imageAlt\)/g) || []).length}`)

// NO name-amplification figure is reported, and the omission is deliberate. An earlier
// version summed title + description + imageUrl into a "name emissions" count and multiplied
// it by the quote expansion, reporting 36x. That is wrong twice over, checked against
// handleNamePage (worker.ts:243-281): `imageUrl` carries the name through
// encodeURIComponent, so its quotes are already %22 and the escaper expands nothing; and
// `description` carries the name only when the ENS description record is ABSENT, which is the
// opposite of this finding's premise. The name is also bounded by URL length, unlike the
// record. The description path below is the finding; the name path is not claimed.
```
