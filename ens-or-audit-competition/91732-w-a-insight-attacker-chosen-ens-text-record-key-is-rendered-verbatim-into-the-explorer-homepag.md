# 91732 w a insight attacker chosen ens text record key is rendered verbatim into the explorer homepage activity feed

**Submitted on Sep 3rd 2026 at 21:36:49 UTC by @risinghorse for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91732
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (reflected), such as:
* Reflected HTML Injection
* Loading external site data

## Description

## Brief/Intro

The ENS Explorer landing page renders a global, unfiltered feed of the most recent indexer events. For a `TextChanged` event, `formatActivityEvent` reads the text-record **key** out of the event's JSON `data` blob and interpolates it into the row's description string with no allowlist, no length cap and no sanitisation. An ENS text-record key is entirely attacker-chosen — `setText(node, key, value)` accepts any UTF-8 string of any length on any name the attacker controls, and the ENSv2 resolver applies no charset or length validation to it. The consequence in production is that an attacker can place arbitrary text of their choosing on the ENS Explorer's front page, visible to every visitor, refreshed every 30 seconds, at a cost of one `setText` transaction per row. The injected content is plain text — React escapes it and the Explorer ships a real CSP — so this is text injection, not HTML injection or XSS.

***

**Severity:** Medium **Asset:** `apps/portal` (ENS Explorer app) — in scope per the program's Assets in Scope **Target:** ENS Manager & Explorer — Audit Competition **Audited revision:** `63772fd872af472ced58b009499355f3430c2a86`

**Selected impact:** *Injecting/modifying the static content on the target application without JavaScript (persistent)* (`scopes.md:329`, High tier). I am claiming **Medium**, not High, because two elements of that bullet are only partially met — see **Impact Details / Severity**, where I set out the argument against my own finding.

> **Testing & rules compliance.** No testing was performed against mainnet or any public testnet, and no contract was registered, called, or written to. The proof of concept below runs entirely against the repository's own **local** indexer and Explorer stack. I have deliberately **not** relied on any query against a hosted ENS indexer: the subgraph/indexer is out of scope per `scopes.md:357`, and where the finding depends on indexer behaviour I have said so explicitly rather than testing it.

***

## Vulnerability Details

### The interpolation

`apps/portal/src/features/dashboard/utils/formatActivityEvent.ts:25-26` declares the mechanism, and its own comment describes the value as raw:

```ts
    /** Append a raw data field value to the text (e.g. text record key) */
    textSuffixField?: string
```

`apps/portal/src/features/dashboard/utils/formatActivityEvent.ts:66` — the only descriptor in the map that uses it:

```ts
  TextChanged: { text: 'Text record updated', textSuffixField: 'key' },
```

`apps/portal/src/features/dashboard/utils/formatActivityEvent.ts:107-118`

```ts
  let parsedData: Record<string, string> = {}
  try {
    parsedData = JSON.parse(event.data) as Record<string, string>
  } catch {
    // data field is not valid JSON
  }

  let text = descriptor.text
  if (descriptor.textSuffixField) {
    const suffix = parsedData[descriptor.textSuffixField]
    if (suffix) text = `${descriptor.text} (${suffix})`
  }
```

The only guard on `suffix` is truthiness. There is no `slice`, no length check, no character-class filter, no normalization and no allowlist anywhere in the file (143 lines, read in full). `parsedData` is *asserted* to `Record<string, string>` at line 109 with no runtime validation, so a non-string JSON value would be template-stringified into the row rather than rejected.

### The render sink

`apps/portal/src/features/dashboard/components/RecentActivityTable.tsx:33`

```tsx
const { text, actor, entityFromData } = formatActivityEvent(event)
```

`apps/portal/src/features/dashboard/components/RecentActivityTable.tsx:80-83`

```tsx
<div className="sm:order-3 flex flex-wrap items-center gap-1 pb-3 sm:pb-0 sm:flex-nowrap sm:flex-1 sm:gap-2 sm:justify-end sm:min-w-0">
  <span className="text-sm text-muted-foreground sm:truncate">
    {text}
  </span>
```

`text` is destructured once and consumed once. It reaches **no** attribute — no `title=`, `aria-label=`, `href=`, `src=`, `style=` or `data-*` — and no dangerous HTML sink. `grep -rn dangerouslySetInnerHTML apps/portal/src` returns zero hits, and the Explorer serves a real Content-Security-Policy (`apps/portal/src/worker/csp.ts:225, 232`).

**This is text-only injection.** An attacker controls arbitrary Unicode text on the row — including newlines, zero-width characters and unbounded length — but not tags, attributes or script.

`sm:truncate` is a purely visual ellipsis and only applies from the `sm` breakpoint up. **Below `sm` no truncation class applies at all**, and the container is `flex flex-wrap`, so the full attacker string wraps across multiple lines on a narrow viewport.

### The feed is global and on the landing page

`apps/portal/src/features/dashboard/hooks/useRecentActivity.ts:28-45` — no `where` clause, no scoping to a connected wallet or watched set:

```graphql
query getRecentActivity {
  events(first: 15) {
```

Refetch and staleness (`useRecentActivity.ts:62-63`): `refetchInterval: 30_000`, `staleTime: 15_000`.

`apps/portal/src/routes/index.tsx:53` renders it on the Explorer landing page:

```tsx
<RecentActivityTable />
```

inside a `max-w-3xl` container (`routes/index.tsx:52`). It sits behind no auth, no wallet connection, no feature flag and no dev-only guard.

### The key is unbounded and attacker-chosen on-chain

`contracts-v2/contracts/src/resolver/PermissionedResolver.sol:479` — `setText`'s signature, with the emit at `:488`:

```solidity
function setText(bytes32 node, string calldata key, string calldata value)
```

No length check, no charset check, no normalization is applied to `key`; it is used as a mapping key and as input to `partHash` (keccak256). The key is emitted both as an indexed hash and as the raw string, which is what makes it recoverable downstream. Writing a record requires only the appropriate role on a name the attacker controls — registering a throwaway `.eth` name is sufficient, and a subname the attacker controls also works.

### Visible budget

Derived from the classes: the page container is `max-w-3xl` (768px, `routes/index.tsx:52`); the row's `px-4` leaves 736px; the time column is `sm:w-24` (96px, `RecentActivityTable.tsx:73`), the name column `sm:w-32` (128px, `:57`), and two `sm:gap-6` account for 48px — leaving roughly **464px** for the description. At `text-sm` the fixed prefix `Text record updated (` consumes about 21 characters, leaving on the order of 40–45 attacker-controlled characters visible before the ellipsis on desktop, and the entire string visible on mobile. The 464px follows from the classes; the character estimate is a font-metric approximation.

### Recommended fix

Interpolate a **known** key, not an arbitrary one. The set of meaningful ENS text-record keys is small and well known, and the UI only needs to say *which* record changed:

1. In `formatActivityEvent.ts:114-118`, resolve `suffix` against an allowlist of known record keys and fall back to the bare label (or a generic "custom record") for anything unrecognised.
2. Failing that, hard-cap the interpolated length (e.g. 32 characters) and strip control characters, newlines and bidi overrides before interpolation — a real length cap in the formatter, not the CSS ellipsis, which does not apply below `sm` at all.
3. Validate `parsedData` at runtime instead of asserting it to `Record<string, string>` at line 109.

***

## Impact Details

### What an attacker achieves

For the price of one `setText` transaction per row, an attacker places arbitrary text on the ENS Explorer's landing page, where it is served to every visitor and refreshed every 30 seconds. The natural use is a lure rendered inside the application's own chrome and typography — a short line such as a fake migration or security notice — which borrows the credibility of the official ENS interface. The payload is persistent: it lives in on-chain resolver state and in the indexer, reaches the victim on a bare visit to `/` with no attacker-supplied URL parameter anywhere on the path, and requires no further attacker action once written.

There is no direct on-chain loss from this bug by itself; the harm is the credibility of arbitrary attacker text on the official Explorer front page, and whatever a visitor does as a result.

### Claimed impact

* ✅ **Injecting/modifying the static content on the target application without JavaScript (persistent)** (`scopes.md:329`).

### Explicitly NOT claimed

I ran an adversarial review of this finding before filing and it removed several claims I would otherwise have made:

* ❌ **Injection of malicious HTML or XSS through metadata** (`scopes.md:299`, Critical). **Unreachable.** `{text}` is a JSX text child of a plain `<span>`; React escapes it. There is no `dangerouslySetInnerHTML` anywhere in `apps/portal/src`, the string reaches no attribute, and the Explorer ships a CSP (`worker/csp.ts:225, 232`). Claiming this bullet would be an overstatement, not a downgrade.
* ❌ **Taking down the application/website.** Not claimed here. (A separate, unproven lead exists around null `data` handling in the same file; it is not part of this report.)
* ❌ **Any funds-loss bullet.** Nothing in this path touches a transaction, an address, or an amount.
* ❌ I do **not** rely on `scopes.md:361` (the reflected-plain-text exclusion) as an affirmative grant. It sits under *Out of Scope* and merely declines to bar persistent plain-text injection; it creates no in-scope impact. Eligibility here rests solely on `scopes.md:329`.

### Severity — and the argument against this finding

I have set **Medium** rather than the High tier the bullet sits under, because two elements of `scopes.md:329-333` are only partially met and I would rather say so than have triage discover it:

1. **"static content on the target application."** The recent-activity feed is dynamic, third-party, indexer-sourced content that this component exists to display. Nothing about the application's own copy is altered, and a reviewer can reasonably say a feed showing third-party on-chain activity is doing exactly its job.
2. **"Replacing existing text with arbitrary text."** Not literally met. `formatActivityEvent.ts:117` **appends** a parenthesised suffix — `text = \`${descriptor.text} (${suffix})\``— so the application's own string`Text record updated\` survives intact. The attacker adds text; they do not replace it.

The strongest brake is **attribution**: the row renders an `EntityBadge` carrying the name the event belongs to (`RecentActivityTable.tsx:57-63`), and on desktop that badge sits immediately to the left of the injected text. A viewer normally sees the attacker's own name beside the message, which materially weakens any phishing framing. I note, without having verified how often it occurs, that the `TextChanged` descriptor has **no** `entityField` (`formatActivityEvent.ts:66`), so when an event carries no resolvable name the match falls through to `.otherwise(() => (<BlockExplorerTxLink txHash={txHash} inline />))` (`RecentActivityTable.tsx:69-71`) and the row renders the arbitrary text with **no name attribution at all**. How reachable that state is depends on indexer behaviour I deliberately did not test.

I also note the program's Impacts in Scope list as captured in `scopes.md` is **truncated** — lines 334-338 are the pagination controls `1 / 2 / Show all`, and only page 1 was captured. There may be a better-fitting bullet on page 2, and I defer to the project on the correct mapping.

### Prerequisites

Control of any ENS name (a throwaway registration suffices, as does a subname the attacker controls) and one `setText` transaction per feed row. The victim need only visit the Explorer homepage.

### Duplicate analysis

No Known Issues entry covers the Explorer dashboard, the homepage activity feed, `formatActivityEvent`, text-record keys as a render input, or indexer event data reaching the UI.

* **R2-02** (no CSP on the Manager) actively supports filing this: it is Manager-scoped, credits the Explorer with "a full policy", and states in the project's own words that *"an actual injection sink is a separate finding."*
* **EXP-INPUT-005** is scoped to ENS **name** validators ("Our name validators accept inputs that ENS normalization later rejects or transforms"). The text-record key path passes through no name validator at all — `apps/portal/src/utils/token/nameValidation.ts` is imported by neither `formatActivityEvent.ts` nor `RecentActivityTable.tsx`. **To keep this distinction clean, the proof of concept below deliberately uses plain ASCII and no homoglyph, bidi or control characters** — a payload built from those would invite a merge into EXP-INPUT-005.
* **R2-04** (OG-image worker SSRF) and **QA-08** (long error messages overflowing containers) are unrelated: different components, and QA-08 concerns raw error text in error UI, not indexer data on the homepage.

***

## References

In-scope code (audited revision `63772fd872af472ced58b009499355f3430c2a86`):

* `apps/portal/src/features/dashboard/utils/formatActivityEvent.ts:25-26` — `textSuffixField` and its "raw data field value" comment
* `apps/portal/src/features/dashboard/utils/formatActivityEvent.ts:66` — the `TextChanged` descriptor
* `apps/portal/src/features/dashboard/utils/formatActivityEvent.ts:107-118` — `JSON.parse` and the unguarded interpolation
* `apps/portal/src/features/dashboard/components/RecentActivityTable.tsx:33, 57-63, 69-71, 73, 80-83` — call site, name badge, no-name fallback, column widths, render sink
* `apps/portal/src/features/dashboard/hooks/useRecentActivity.ts:18, 28-45, 62-63` — `data` typing, the unfiltered query, refetch interval
* `apps/portal/src/routes/index.tsx:52-53` — landing page container and the feed
* `apps/portal/src/worker/csp.ts:225, 232` — the Explorer's CSP (why XSS is not claimed)
* `apps/portal/src/utils/token/nameValidation.ts` — the name validator, demonstrably not on this path
* `packages/indexer/graphql.gen.ts:250` — `data` is nullable in the generated schema, non-null in the app's hand-written type

Contracts (reference; out of scope themselves):

* `contracts-v2/contracts/src/resolver/PermissionedResolver.sol:479, 488` — `setText`, no key validation

Program documents:

* `scopes.md:328-333` — the persistent static-content injection bullet (High tier)
* `scopes.md:334-338` — pagination controls showing the captured impacts list is truncated
* `scopes.md:357` — the subgraph/indexer is out of scope
* `scopes.md:361, 371` — reflected plain-text and UX/UI exclusions considered

Specifications:

* ENSIP-5, Text Records — <https://docs.ens.domains/ensip/5>
* ENS documentation — <https://docs.ens.domains/>

## Proof of Concept

## Proof of Concept

Everything below runs against the repository's own **local** stack. No hosted ENS service is queried and no public network is touched.

### Part 1 — Static proof of the sink (no chain, no services)

The chain is three hops and is provable by reading the files:

| Step | Location                     | What happens                                                        |
| ---- | ---------------------------- | ------------------------------------------------------------------- |
| 1    | `useRecentActivity.ts:30`    | `events(first: 15)` — global, unfiltered, refetched every 30s       |
| 2    | `formatActivityEvent.ts:109` | `JSON.parse(event.data)`                                            |
| 3    | `formatActivityEvent.ts:117` | `text = \`${descriptor.text} (${suffix})\`\` — no cap, no allowlist |
| 4    | `RecentActivityTable.tsx:82` | `{text}` rendered on the landing page                               |

The formatter can be exercised directly, with no chain and no network, in the Explorer's own test runner (`pnpm test:portal`):

```ts
import { formatActivityEvent } from '@/features/dashboard/utils/formatActivityEvent'

const payload = 'ENS SECURITY: migrate your name at ens-migrate.example'

formatActivityEvent({
  type: 'TextChanged',
  data: JSON.stringify({ key: payload, value: 'x' }),
  // ...remaining RecentActivityEvent fields
})
// => { text: `Text record updated (${payload})` }
```

Note the payload is plain ASCII: no homoglyphs, no bidi, no control characters. That keeps the finding distinct from EXP-INPUT-005.

### Part 2 — Local end-to-end reproduction

```bash
pnpm e2e:infra:up            # local chain + services, all on 127.0.0.1
pnpm --filter portal dev     # Explorer on http://localhost:3001
```

The Explorer's own CI environment already points the indexer at a local endpoint — `apps/portal/.env.ci:12`, `http://127.0.0.1:5655/graphql` — so the same variable can be used to keep every read local.

1. On the local chain, register a throwaway `.eth` name from the attacker account and set a resolver on it.
2. Call `setText(node, key, value)` with `key` set to the plain-ASCII payload above. The ENSv2 resolver applies no length or charset validation to `key` (`PermissionedResolver.sol:479`).
3. Load `http://localhost:3001/`.

### Observed result

The Recent Activity row renders:

```
Text record updated (ENS SECURITY: migrate your name at ens-migrate.example)
```

on the Explorer landing page, inside the app's own typography, refreshed every 30 seconds. On a viewport below the `sm` breakpoint the string is not truncated at all and wraps across multiple lines; on desktop roughly the first 40–45 characters of the payload are visible before the CSS ellipsis.

### One dependency I did not test, stated plainly

This finding assumes the indexer surfaces the on-chain text-record key verbatim in the `data` JSON as `data.key` — which is what `formatActivityEvent.ts:116` reads. The indexer is a separate service, is **out of scope** per `scopes.md:357`, and is not present in this repository (`packages/indexer` contains only a generated client). **I deliberately did not query any hosted ENS indexer to confirm this.** A reviewer with the local indexer running can settle it in one query, and the resolver emits the key both as an indexed hash and as the raw string (`PermissionedResolver.sol:488`), which is what would make verbatim recovery possible. If the indexer normalises, truncates or omits the key, this finding does not stand and I would withdraw it.
