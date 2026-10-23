# 92816 w a medium the expiry discovery cron s non unique timestamp cursor pagination permanently and silently drops names tied at a page 1000 boundary from every expiry notification

**Submitted on Sep 12th 2026 at 14:32:55 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92816
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Summary

The hourly expiry-discovery cron (`runExpiryDiscoveryCron`, `index.ts:150-240`) walks four notification stages (`30d`/`7d`/`1d`/`expired`, `stages.ts:9-30`) and, per stage, pages through the ENS subgraph with a **strict, non-unique-key** cursor:

```graphql
domains(where: { expiry_gt: $cursor, expiry_lte: $upper_bound }, orderBy: expiryDate,
        orderDirection: asc, first: 1000)
```

(`indexer.ts:13-28`, `PAGE_SIZE = 1000` at `indexer.ts:9`). After each page, `processStage` advances the persisted cursor to the **last** returned domain's `expiryDate` (`index.ts:136`: `cursorEnd: lastExpiryDate ?? ctx.cursor`), and the next run's query uses `expiry_gt: cursorEnd` — strictly greater than the value just stored.

`expiryDate` is not a unique key: any set of domains registered or renewed in the **same block** with the **same duration** get byte-identical `expiryDate` values. This is a normal, expected occurrence in this exact system, not a contrived edge case — this codebase itself ships a bulk-renewal feature (`apps/manager/src/features/bulk-renew`) that renews many names to the same new expiry in one transaction, and Sepolia's short block time means ordinary registration bursts routinely land in the same block. If a full 1000-row page boundary falls in the **middle** of a group of domains sharing one exact `expiryDate`, the domains at that exact value that did not make it into the page are **permanently and silently excluded from every future page**, because `expiry_gt: cursorEnd` can never again match that value — the cursor only ever advances forward, and there is no separate "catch stragglers" pass. Those domains never receive any of the four expiry-notification stages, while the name's own on-chain expiry, grace period, and in-app dashboard/profile display remain completely correct and unaffected — this is confined entirely to the notification side-channel.

I built a real PoC proving this against the actual, unmodified service code: a synthetic dataset where 5 domains share one exact `expiryDate` straddling the real `PAGE_SIZE = 1000` boundary (3 of the 5 land in page 1, 2 do not). Across 5 simulated cron ticks, the 2 leftover domains (`boundary-4.eth`, `boundary-5.eth`) are returned by the subgraph query **zero** times after tick 1 and never once reach the notification-queue publish call (`EVENT_INGESTION_QUEUE.sendBatch`), while every other domain in the dataset — including domains immediately before and after the boundary — is correctly enqueued exactly once.

## Finding Description

### Intended behavior

The expiry-discovery cron is meant to page through *every* domain whose expiry falls inside each stage's window exactly once, so that every domain (and, for the later three stages, every user who favourited it) receives its 30-day, 7-day, 1-day, and expired notification. A page-based cursor over a subgraph query is a reasonable, standard implementation of that — provided the sort key used for the cursor cut is unique, or the cursor logic otherwise accounts for ties at the cut point (e.g. a compound `(expiryDate, id)` cursor, or re-querying inclusively and de-duplicating already-seen ids).

### Actual behavior

#### 1. The cursor is keyed purely on `expiryDate`, a non-unique field

`indexer.ts:13-28`:

```ts
const expiringNamesQuery = gql`
  query GetExpiringNames($cursor: Int!, $upper_bound: Int!) {
    domains(
      where: { expiry_gt: $cursor, expiry_lte: $upper_bound }
      orderBy: expiryDate
      orderDirection: asc
      first: ${PAGE_SIZE}
    ) {
      name
      expiryDate
      owner {
        id
      }
    }
  }
`
```

No secondary sort key (such as `id` or `name`) is specified anywhere in this query. `PAGE_SIZE` is a fixed `1000` (`indexer.ts:9`).

#### 2. The cursor advances to the last row's value, not past every tied row

`index.ts:109-147` (`processStage`):

```ts
const events = buildExpiryEvents(ctx.stage, page.domains)
const eventChunks = chunk(events, QUEUE_BATCH_SIZE)
const firstExpiryDate = page.domains[0]?.expiryDate
const lastExpiryDate = page.domains[page.domains.length - 1]?.expiryDate
...
return ok({
  stageId: ctx.stage.id,
  cursorStart: ctx.cursor,
  cursorEnd: lastExpiryDate ?? ctx.cursor,
  upperBound,
  enqueuedCount: events.length,
  pageDomainCount: page.domains.length,
  chunkCount: eventChunks.length,
  hasMore: page.hasMore,
  firstExpiryDate,
  lastExpiryDate,
} satisfies StageRunMetrics & { firstExpiryDate?: number; lastExpiryDate?: number })
```

If the 1000th (last) row's `expiryDate` is shared by one or more domains that did **not** make it into this page — because the query only ever asked for `first: 1000` and the tie-break among equal-`expiryDate` rows is the subgraph's own internal ordering, not something this client controls or even observes — `cursorEnd` is still set to that shared value.

#### 3. The next query's `expiry_gt` permanently excludes the shared value

`runExpiryDiscoveryCron` persists `cursorEnd` via `storeNotificationCursors` (`index.ts:224`, `cursors.ts:92-102`) and the next run's `processStage` call reads it back as `ctx.cursor` (`index.ts:170-175`), which flows straight into the next `fetchExpiringNamesPage` call's `cursor` variable (`index.ts:84-89`) and from there into the query's `expiry_gt: $cursor` (`indexer.ts:14-16`). `expiry_gt` is strict: a domain whose `expiryDate` exactly equals the stored cursor can never again satisfy `expiry_gt: cursor`, on this run or any future run, since the cursor for this stage only ever moves forward (`index.ts:59-75` returns early once `ctx.cursor >= upperBound`, and nothing ever moves it backward).

#### 4. No downstream consumer can recover a dropped domain

`buildExpiryEvents` (`index.ts:35-47`) maps each *returned* domain 1:1 into an `ExpiryEvent`; a domain that the indexer query never returns simply never produces an event — there is no zip/join by index anywhere in this pipeline that a drop could desynchronize into a wrong pairing, and no separate reconciliation pass anywhere in `workers/api-worker` re-scans a stage's already-passed timestamp range. Each of the four stages tracks its own independent cursor (`cursors.ts:12-17`), so a drop in one stage does not, by itself, affect the other three stages' independent windows — but since the underlying `expiryDate` value is identical across all of a victim's tied siblings, the exact same page-boundary collision is equally likely to recur independently in each stage's own pagination as that stage's cursor sweeps past the same timestamp.

### The value/state edge, restated precisely

* Page returns **fewer than 1000 domains**: no truncation occurs mid-timestamp-group; every domain actually seen is included, and `cursorEnd` is set past all of them. **Safe.**
* Page returns **exactly 1000 domains** and the 1000th (last) domain's `expiryDate` is shared by one or more domains **not** returned on this page: those un-returned domains are permanently and silently dropped from this stage's notification pipeline, on the very next query.
* **Duplicate count exactly at the boundary** (e.g. one tied domain returned, one not): identical outcome — the withheld one is lost forever, not merely delayed to the next run, which is exactly what this PoC demonstrates across 5 simulated cron ticks.
* **Empty page** (`page.domains.length === 0`, `index.ts:91-107`): handled correctly — the cursor is left unchanged and the stage naturally retries the same window on the next run. Not part of this defect.

## Root Cause

`fetchExpiringNamesPage`'s subgraph query (`indexer.ts:13-28`) sorts and pages strictly by `expiryDate` with no secondary/tie-breaking key, and `processStage` (`index.ts:136`) advances the persisted cursor to that field's own value from the last row of the page. Because `expiryDate` is not unique, this is a `(cursor, ∞)`-style exclusive cut over a field that legitimately repeats across many rows; any page boundary that lands inside a tied group permanently orphans the withheld rows behind an `expiry_gt` filter that can never again match them. The fix requires either a compound cursor (`(expiryDate, id)` with a matching compound filter/sort in the subgraph query) or re-querying the boundary value inclusively each time and de-duplicating already-enqueued ids client-side.

## Impact Explanation

**Mechanics, proven by the PoC** (independent, from-scratch re-run; full evidence in `poc/apiworker-expiry-discovery-cursor-gap/evidence/run-output.json`):

```
Synthetic dataset: 997 domains with unique expiryDate values, then 5 domains
("boundary-1.eth".."boundary-5.eth") sharing ONE exact expiryDate = 1739223252,
then 1 domain ("after-boundary.eth") at a later, unique expiryDate = 1739224252.

Tick 1 (first page, cursor=1739222152 [seeded], upper_bound=1789222252):
  matchedCount: 1003, returnedCount: 1000
  returned: uniq-1.eth .. uniq-997.eth, boundary-1.eth, boundary-2.eth, boundary-3.eth
  enqueued to EVENT_INGESTION_QUEUE.sendBatch: 1000 events
  cursorEnd persisted: 1739223252   (== the shared boundary expiryDate)

Tick 2 (cursor=1739223252):
  matchedCount: 1, returnedCount: 1
  returned: after-boundary.eth   <- correctly returned (unique, later expiryDate)
  enqueued: 1 event
  cursorEnd persisted: 1739224252
  boundary-4.eth / boundary-5.eth: NOT returned, NOT enqueued

Ticks 3, 4, 5 (cursor=1739224252, unchanged):
  matchedCount: 0, returnedCount: 0 each tick
  boundary-4.eth / boundary-5.eth: still NOT returned

Across all 5 ticks / all 5 subgraph requests:
  "neverAgainReturned": { "boundary-4.eth": 0, "boundary-5.eth": 0 }
```

`boundary-4.eth` and `boundary-5.eth` are present in the dataset for the entire run (confirmed directly: `syntheticDataset.map(d => d.name)` contains both), yet the real, unmodified `fetchExpiringNamesPage`/`processStage` code never returns or enqueues either of them again after tick 1, across 5 consecutive cron cycles — the maximum a hostile query outcome could be "eventually retried later" is disproven directly, in favor of "provably permanent" for as long as the cursor never regresses (it never does).

**Upgrade-hunt (why this stays a Medium, not a Critical/High):**

* **Fund-loss path:** none. The dropped domain's own dashboard/profile page (which reads on-chain expiry/grace data independently of this notification pipeline) continues to show the correct real expiry at all times; nothing about the registration, its grace period, or its ownership is touched. The only thing that silently fails is a best-effort, off-chain reminder channel.
* **Griefing-for-profit (an attacker suppresses a&#x20;*****specific*****&#x20;victim's reminders ahead of sniping their lapsed name):** requires forcing the victim's own, already-fixed `expiryDate` to land exactly at a page cutoff with extra same-timestamp filler entries beyond it, sorted ahead of the victim's row by whatever internal tie-break the subgraph applies — a tie-break this client never observes or controls, so which of several tied rows survives the cut is not reliably steerable toward excluding a specific target. This does not clear a credible High bar.
* **Mass-availability angle (an attacker cheaply suppresses reminders for many arbitrary users at once):** registering/renewing ≥1000 names sharing one exact `expiryDate` guarantees a real page-cutoff split, at the attacker's own real registration cost. The ceiling is still "some users don't get an email/push/Telegram reminder they'd otherwise have gotten" — no fund transfer, no wrong balance, no unauthorized access, and the underlying on-chain expiry/grace state and every in-app display remain fully correct and independently visible. This keeps the finding at Medium.
* **Cross-user reach:** for the `7d`/`1d`/`expired` stages, `includeFavorites: true` (`stages.ts:18-29`) means any user who favourited the affected name — not just its owner — silently stops receiving reminders for it, which is exactly the "other user" framing the cited Medium bucket describes.

This is why the taxonomy's "Enabling/disabling notifications" Medium example is the correct, honest fit: the observable end-state for affected users is indistinguishable from having their expiry notifications silently disabled for one specific name, with no error, banner, or signal anywhere that this happened.

## Likelihood Explanation

No attacker action is required at all for this to occur naturally and repeatedly:

1. **Same-block registration/renewal bursts** are an ordinary occurrence for any EVM chain with short block times (Sepolia here), and become exactly-identical `expiryDate` groups whenever two or more registrations/renewals in the same block use the same duration — a common choice, since UIs typically default to whole-year durations.
2. **This codebase's own bulk-renewal feature** (`apps/manager/src/features/bulk-renew`) renews an arbitrary number of names to the **same new expiry** in a single transaction by design — this is the single most direct, first-party, in-scope source of large exact-`expiryDate` groups, and requires zero external tooling or malicious intent from the renewing user.
3. Whether any such group happens to straddle the exact 1000-row page cutoff for a given stage on a given hourly run is a function of how many *other*, unrelated domains also fall in that stage's current window — over the lifetime of a production deployment with any meaningful registration volume, some 1000-row page boundary landing inside a same-block/same-duration group is a matter of *when*, not *if*, and it recurs independently for every stage that later sweeps past that same timestamp.
4. Separately (see Impact Explanation), an attacker could deliberately manufacture this condition cheaply and at will for **many arbitrary victims simultaneously** (not a specific chosen victim) by registering/renewing ≥1000 names sharing one exact `expiryDate` — this does not require any interaction with, or privilege over, any other user's wallet, session, or data.

Because the defect requires no click, no already-connected-wallet interaction from any victim, and recurs on ordinary, first-party application behavior (bulk-renew) with zero attacker involvement, likelihood is realistic and non-speculative, matching the "up to one click of user interaction" (here: zero) framing of the cited Medium bucket.

## Trigger Path / Reproduction Path

This is not an attacker-triggered exploit chain — it is a naturally occurring data condition (a page-boundary landing inside a group of domains that legitimately share one exact `expiryDate`) combined with a pagination/cursor logic defect. The path below is the natural trigger sequence, reproduced deterministically in the PoC:

1. Over time, ≥1000 domains accumulate inside one expiry-discovery stage's current window (`(cursor, upperBound]`), such that a group of domains sharing one exact `expiryDate` value (from same-block registrations/renewals with the same duration, or from a single bulk-renewal transaction) straddles the 1000th row when sorted by `expiryDate` ascending.
2. The hourly cron runs `fetchExpiringNamesPage`, which returns exactly the first 1000 rows — some, but not all, of the tied-`expiryDate` group.
3. `processStage` sets the persisted cursor to the last returned row's `expiryDate`, i.e. the shared, tied value.
4. On the next hourly run (or any run thereafter), the query's `expiry_gt: cursor` excludes every row at that exact value, including the domains that were withheld from the first page.
5. Those domains never appear in any subsequent page, for this stage, ever again — they silently never receive that stage's notification, indefinitely, with the underlying on-chain expiry, grace period, and dashboard/profile display completely unaffected and still fully correct.

**PoC reproduction (deterministic, no live chain/subgraph needed):** a synthetic dataset is built with exactly this shape (see PROOF OF CONCEPT below) and the real, unmodified `runExpiryDiscoveryCron` is invoked 5 times in a row against a local mock subgraph server serving that dataset, proving steps 2-5 exactly as described above.

## External Preconditions

* A stage's current expiry window (`(cursor, upperBound]`) contains at least 1000 domains total when queried at some cron tick.
* At least one group of domains inside that window shares one exact `expiryDate` value that straddles the 1000th-row cutoff when the subgraph's own internal tie-break order is applied — a condition this application itself organically produces via same-block registrations/renewals or its own bulk-renewal feature, and which an attacker can also cheaply and deliberately manufacture (registering/renewing ≥1000 names at one shared `expiryDate`) without needing any interaction with any victim's wallet, session, or account.

## Internal Preconditions

* `fetchExpiringNamesPage`'s subgraph query (`indexer.ts:13-28`) continues to sort/filter purely by `expiryDate` with no secondary/unique tie-breaking key.
* `processStage` (`index.ts:136`) continues to advance the persisted cursor to the last returned row's `expiryDate` without any inclusive re-check or de-duplication of ties at that exact value.

## Affected Code

* [`workers/api-worker/src/services/expiry-discovery/indexer.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/indexer.ts#L9-L28) — `PAGE_SIZE = 1000` (line 9) and the `expiringNamesQuery` (lines 13-28), sorted/filtered purely by the non-unique `expiryDate` field.
* [`workers/api-worker/src/services/expiry-discovery/index.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/index.ts#L84-L148) — `processStage`, which fetches one page and advances the cursor to `lastExpiryDate` (line 136) with no tie-awareness.
* [`workers/api-worker/src/services/expiry-discovery/index.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/index.ts#L150-L240) — `runExpiryDiscoveryCron`, which persists the advanced cursor per stage (lines 204-224) so the drop is permanent across runs.
* [`workers/api-worker/src/services/expiry-discovery/cursors.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/cursors.ts) — cursor persistence itself is correct; included for completeness since it is what makes the drop durable across cron invocations.
* [`workers/api-worker/src/services/expiry-discovery/stages.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/stages.ts#L9-L30) — `STAGES`, showing `includeFavorites: true` for `7d`/`1d`/`expired`, which is why the affected party is not limited to the domain's own owner.

## Recommended Mitigation

Replace the single-field, non-unique cursor with a compound `(expiryDate, id)` (or `(expiryDate, name)`) cursor: sort the subgraph query by `orderBy: [expiryDate, id]` (or equivalent), filter with `(expiryDate > $cursorExpiry) OR (expiryDate == $cursorExpiry AND id > $cursorId)`, and persist both the last row's `expiryDate` and `id` as the stage cursor. This makes the cut point unique per row regardless of how many domains share the same `expiryDate`, so a page boundary can never again land "inside" a tied group in a way that permanently orphans rows. If a compound filter is not readily expressible against the current subgraph schema, an inclusive re-query of the exact boundary value each time (`expiry_gte: cursor`) combined with client-side de-duplication against a small set of already-enqueued `(name, expiryDate)` pairs from the previous page achieves the same correctness at the cost of a small amount of redundant work per page.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/9443d20a49dc8d09efa4ce20bcc739d3/e5f450d4f9cd2733722dc5eba24914d428312e18>

## Proof of Concept

**Description**

This PoC runs the real, unmodified `runExpiryDiscoveryCron` / `processStage` / `fetchExpiringNamesPage` chain from `workers/api-worker/src/services/expiry-discovery/{index.ts, indexer.ts}` against a 100%-local mock GraphQL subgraph HTTP server (a plain `node:http` server bound to `127.0.0.1` on an OS-assigned ephemeral port — no external network access anywhere in this PoC). The repository's own `expiry-discovery/*.test.ts` files run under `@cloudflare/vitest-pool-workers` (a real `workerd` sandbox that cannot open a local TCP listening socket), so this PoC instead uses a standalone, plain-Node `vitest.config.ts` (only for this PoC directory) that maps the package's own `"imports": { "#*": "./src/*" }` subpath-import convention so the unmodified source files' own `#utils/...`/`#core/...`/`#types/...` imports resolve unchanged — this is purely a test-harness choice; it does not alter, stub, or wrap a single line of `index.ts`/`indexer.ts`.

A synthetic dataset of 1003 domains is served by the mock server: 997 with unique, ascending `expiryDate`s, then 5 domains (`boundary-1..5.eth`) sharing **one exact** `expiryDate` value positioned so it straddles the real `PAGE_SIZE = 1000` boundary (3 of the 5 land inside page 1, 2 do not), then 1 domain (`after-boundary.eth`) with a later, unique `expiryDate`. The real cron is invoked 5 times in sequence against the same persisted `env.KV` cursor store (simulating 5 real hourly cron ticks), and `env.EVENT_INGESTION_QUEUE.sendBatch` is instrumented to record every notification-triggering call the real code makes.

**Where to place files and their names**

Place the complete PoC under the frozen repository root:

```
poc/apiworker-expiry-discovery-cursor-gap/
├── README.md
├── SOURCE_SHA256SUMS
├── .gitignore
├── vitest.config.ts
├── mock-subgraph-server.mjs
├── cursor-gap.spec.ts
├── run.sh
└── evidence/
    ├── run-output.json
    └── run-output.txt
```

**Tool and dependency versions**

```
Node.js: 24.18.0
pnpm:    10.27.0
Vitest:  4.1.5 (the repository's own installed vitest binary, invoked via a
         PoC-local, plain-Node vitest.config.ts)
```

**PoC Link**

```
https://gist.github.com/aydarkhusaenov/9443d20a49dc8d09efa4ce20bcc739d3/e5f450d4f9cd2733722dc5eba24914d428312e18
```

Note: GitHub Gist's git backend does not support subdirectories (a real, confirmed platform constraint hit while pushing this PoC). The gist's `evidence/run-output.json` and `evidence/run-output.txt` are therefore stored there as flattened `evidence__run-output.json` / `evidence__run-output.txt`; rename them back to `evidence/<name>` after cloning to match the layout above (the gist's own `README.md` documents this same mapping).

**Clean clone, checkout, and installation**

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-audit-comp
cd ens-audit-comp
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

mkdir -p poc/apiworker-expiry-discovery-cursor-gap
git clone https://gist.github.com/aydarkhusaenov/9443d20a49dc8d09efa4ce20bcc739d3.git \
  poc/apiworker-expiry-discovery-cursor-gap
cd poc/apiworker-expiry-discovery-cursor-gap
git checkout --detach e5f450d4f9cd2733722dc5eba24914d428312e18
mkdir -p evidence
mv evidence__run-output.json evidence/run-output.json
mv evidence__run-output.txt evidence/run-output.txt
cd ../..

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
```

**Exact run command**

From the frozen repository root:

```bash
bash poc/apiworker-expiry-discovery-cursor-gap/run.sh
```

The runner prints the sha256 of the exact real source files under test, runs the PoC via the repository's own installed `vitest` binary (`workers/api-worker/node_modules/.bin/vitest`), and writes full evidence to `evidence/run-output.txt` (raw console/vitest output) and `evidence/run-output.json` (structured per-tick cursor/enqueue data plus the mock server's full request log). A full run completes in well under one second.

**Trigger actions (what the PoC's synthetic dataset represents)**

1. 997 domains with unique, ascending `expiryDate` values accumulate in the `expired` stage's window (representing ordinary registration history).
2. 5 domains (`boundary-1.eth` .. `boundary-5.eth`) share one exact `expiryDate` (`1739223252` in this run) — representing either a same-block registration/renewal burst with a shared duration, or this application's own bulk-renewal feature renewing 5 names to the same new expiry.
3. This group is positioned so that, sorted by `expiryDate` ascending, it occupies rows 998-1002 of the dataset — straddling the real `PAGE_SIZE = 1000` cutoff exactly between rows 1000 and 1001.
4. 1 further domain (`after-boundary.eth`) sits at a later, unique `expiryDate` (`1739224252`), to prove domains on the far side of the boundary are handled correctly.

**Exploit/reproduction actions**

1. `env.KV` is seeded so the `30d`/`7d`/`1d` stages are already caught up (isolating the experiment to the `expired` stage, the same "cursor already caught up" pattern this package's own test suite already uses) and the `expired` stage's cursor is seeded just before the dataset's earliest timestamp.
2. `env.ENS_INDEXER_GRAPHQL_URL` is set to the local mock server's URL.
3. `runExpiryDiscoveryCron(env)` — the real, unmodified function — is invoked and awaited 5 times in sequence, each call reading/writing the same `env.KV` cursor state, simulating 5 real hourly cron ticks.
4. After each tick, the newly-recorded `EVENT_INGESTION_QUEUE.sendBatch` calls (the real notification-triggering boundary) and the persisted cursor value are captured.
5. Separately and independently, every raw page the mock subgraph server ever returned (across all 5 ticks) is inspected directly from the server's own request log, with no dependency on the notification-queue instrumentation.

**Fresh logs proving the bug**

The clean, independently re-run `run.sh` (fresh evidence directory, fresh mock server instance, fresh vitest process — Step 9 of this program's own methodology; single attempt needed, passed on both the initial build and the independent re-run) produced:

```
Test Files  1 passed (1)
     Tests  1 passed (1)
```

and, from the structured evidence (`evidence/run-output.json`, reproduced verbatim from a real run):

```json
{
  "capturedNowSec": 1789222252,
  "EPOCH": 1739222252,
  "BOUNDARY_EXPIRY": 1739223252,
  "AFTER_BOUNDARY_EXPIRY": 1739224252,
  "datasetSize": 1003,
  "perTick": [
    { "tick": 1, "cursorEndAfterTick": 1739223252, "enqueuedCount": 1000 },
    { "tick": 2, "cursorEndAfterTick": 1739224252, "enqueuedCount": 1 },
    { "tick": 3, "cursorEndAfterTick": 1739224252, "enqueuedCount": 0 },
    { "tick": 4, "cursorEndAfterTick": 1739224252, "enqueuedCount": 0 },
    { "tick": 5, "cursorEndAfterTick": 1739224252, "enqueuedCount": 0 }
  ],
  "neverAgainReturned": {
    "boundary-4.eth": 0,
    "boundary-5.eth": 0
  }
}
```

(Full per-tick enqueued-name lists and the complete 5-request subgraph request log are in the evidence file; truncated here for brevity — every occurrence of `boundary-1.eth`/`boundary-2.eth`/ `boundary-3.eth` (tick 1) and `after-boundary.eth` (tick 2) is present exactly once, confirming correct behavior on both sides of the boundary, while `boundary-4.eth`/`boundary-5.eth` never appear in any tick's enqueued names or in any of the 5 raw subgraph responses.)

Tick-by-tick reading of the log:

* **Tick 1** — subgraph query `expiry_gt: 1739222152` (seeded cursor) returns exactly 1000 rows: the 997 unique domains plus `boundary-1.eth`, `boundary-2.eth`, `boundary-3.eth`. All 1000 are enqueued to `EVENT_INGESTION_QUEUE.sendBatch`. The persisted cursor becomes `1739223252` — the shared boundary timestamp.
* **Tick 2** — subgraph query `expiry_gt: 1739223252` (now excludes the boundary value entirely, including the two leftover tied domains) returns exactly 1 row: `after-boundary.eth`. It is correctly enqueued. The persisted cursor becomes `1739224252`.
* **Ticks 3-5** — subgraph query `expiry_gt: 1739224252` returns 0 rows each time; the cursor stays frozen at `1739224252` forever, since nothing in the dataset exceeds it.
* Across all 5 requests logged by the mock server itself, `boundary-4.eth` and `boundary-5.eth` appear in **zero** returned pages, despite being present in the server's dataset the entire time — proving the drop is not "delayed to next run" but structurally permanent for as long as the cursor never regresses.

**Generated evidence files**

```
evidence/run-output.txt   (full vitest console output, including the sha256 of the exact
                           real source files exercised)
evidence/run-output.json  (structured per-tick cursor/enqueue data and the mock subgraph
                           server's complete 5-request log)
SOURCE_SHA256SUMS         (sha256 of index.ts/indexer.ts/stages.ts/cursors.ts as exercised,
                           captured fresh on each run)
```

**Cleanup**

`run.sh` does not spawn any persistent process, container, or global state — the mock HTTP server is started and closed within the single vitest process invocation (`beforeAll`/`afterAll` in `cursor-gap.spec.ts`), and nothing under `workers/api-worker/src` is ever modified. No cleanup step is required; simply deleting the `poc/apiworker-expiry-discovery-cursor-gap/` directory removes everything this PoC created.
