# 92719 w a insight a gifted ens name with a crafted label injects persistent attacker controlled markdown into a victim s real telegram expiry alert notifications

**Submitted on Sep 11th 2026 at 20:05:56 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92719
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

`workers/api-worker/src/services/delivery/templates/telegram.ts` builds the outbound text for the `name-expiry` Telegram notification by interpolating `payload.name` — an ENS name — directly into a template literal, with zero escaping, inside a message whose `parse_mode` is Telegram's legacy `Markdown` (V1):

```ts
const ownerText = payload.isOwner
  ? `Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\nDon't forget to renew your domain!`
  : `The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.`
```

Nothing in the call chain from there to Telegram's `sendMessage` API — `services/delivery/telegram.ts`, `services/telegram/utils.ts` — escapes or sanitizes this string. Because legacy Markdown pairs each opening special character (`` ` ``, `*`, `[`) with the next unescaped occurrence of the same character, an ENS label containing those characters lets an attacker close the template's own code span early and inject arbitrary bold text and a clickable link with attacker-chosen display text into a message the victim receives from ENS's own real Telegram bot chat.

The ENS on-chain registrar contracts enforce no ENSIP-15 / character-class validation on a label (only client UIs do), so an attacker can register an arbitrary raw-byte label directly on-chain, then **gift the resulting ERC-1155 name token straight to a victim's wallet address** — an ordinary `safeTransferFrom` that requires zero interaction or consent from the recipient. `workers/api-worker`'s hourly expiry-discovery cron resolves the `owned`-notification recipient for a name via a raw address lookup (`event-ingestion.ts`'s `collectRecipientsForEvent`) that never touches any Manager frontend validation code, so it is completely unaffected by the client-side character filters that gate other parts of the app (see "Root Cause" for why the alternative "victim favorites the name" path is dead and this gift/owner path is the one that actually fires). If the victim already has a verified Telegram channel and the shipped `owned_name_expiry` notification toggle enabled — a real, opt-in setting that the app's own post-registration onboarding screen defaults **on** for any user who has just registered or renewed a name of their own — the victim receives, with zero further action from anyone, a message in the same bot chat they already trust for legitimate ENS alerts, carrying attacker-controlled bold "official-looking" text and a phishing link with deceptive display text.

The same template also builds the message's inline-keyboard "Renew Now" button from the identical raw `payload.name`, so the resulting button `url` field contains the same backticks/asterisks/brackets/newlines as the message text. I confirmed from Telegram's own published server source (see Finding Description step 7) that this does **not** cause the whole `sendMessage` call to be rejected — Telegram's real URL validator only restricts the scheme and host of a button URL and silently percent-encodes control bytes/spaces in the path, so the message is delivered regardless; only the button's own destination becomes a non-functional, garbled URL, which does not weaken the primary text-injection finding.

I built a real, non-mocked, end-to-end proof of this exact mechanism (see "Proof of Concept"). It registers a crafted label directly against the real deployed V2 `ETHRegistrar` on a local Anvil fork of Sepolia (`register()` succeeded, tx `0x18b9f8d7dd43dce08fdee4b62fdae5ec04bb5d8a4a7ed48f37279cd0c92e6e1e`), transfers the resulting name to a fresh victim address via a real ERC-1155 `safeTransferFrom` (tx `0xd6ebe988aee3362f263a4e88b00913e430e284efb1b4dd85f2b6b01d3e5254ed`), seeds the real Postgres database with the victim's `owned_name_expiry: true` account state, and drives the real, unmodified `runExpiryDiscoveryCron` → `handleEventIngestionQueue` → `handleTelegramQueue` → `deliverTelegramNotification` → `telegramTemplates['name-expiry']` → `makeTelegramRequest` call chain end to end. The real production code built and POSTed:

```json
{
  "chat_id": "918273645",
  "parse_mode": "Markdown",
  "text": "⚠️ *Domain Expiration Alert*\n\nYour domain `poc-tg-mdinj-1789154480`*🔒 ENS Security Alert:* Your wallet requires immediate re-verification\n\n[Verify your wallet now — official ENS](https://ens-domains-secure-verify.example/action)\n\n\n\n\n\n`.eth` will expire in *3 days*.\n\nDon't forget to renew your domain!"
}
```

which, under Telegram's own documented legacy Markdown parsing rules, renders as a bold fake security alert plus a clickable, attacker-labeled phishing link — inside the real `⚠️ *Domain Expiration Alert*` header the victim already recognizes as legitimate.

## Finding Description

### Intended behavior

The `name-expiry` Telegram notification should present the ENS name a user owns or watches as inert, literal text. An attacker-chosen name should never be able to alter the notification's formatting, insert additional bold/link markup, or cause the message to display attacker-controlled call-to-action text or URLs. The only interactive element in the message should be the fixed "🔄 Renew Now" button the template itself defines.

### Actual behavior

#### 1. The vulnerable sink: `payload.name` interpolated raw into a `parse_mode: 'Markdown'` string

[`workers/api-worker/src/services/delivery/templates/telegram.ts#L28-L49`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/delivery/templates/telegram.ts#L28-L49) (`telegramTemplates['name-expiry']`):

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
    buttons: [[{ text: '🔄 Renew Now', url: `https://app.ens.domains/${payload.name}` }]],
  }
},
```

`payload.name` is substituted with zero escaping between a literal backtick pair, in a string that is later sent with `parse_mode: 'Markdown'`. `daysLeft` is a computed number (not attacker-influenced); the button URL's `https://app.ens.domains/` prefix is a fixed literal, but its path suffix is the same unescaped `payload.name` (addressed in step 7 below).

#### 2. Nothing downstream escapes it before the real Telegram API call

[`workers/api-worker/src/services/delivery/telegram.ts#L23-L92`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/delivery/telegram.ts#L23-L92) (`deliverTelegramNotification`) reads the stored `payload` straight from Postgres, calls the template, and forwards the result verbatim:

```ts
const message = template(
  deliveryJob.notification.payload as AnyPersonalNotificationPayload,
)                                                                              // L58-60
...
const result = yield* makeTelegramRequest(botToken, 'sendMessage', {
  chat_id: deliveryJob.target,
  text: message.text,
  parse_mode: message.parseMode,
  reply_markup: replyMarkup,
})                                                                             // L68-73
```

[`workers/api-worker/src/services/telegram/utils.ts#L33`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/telegram/utils.ts#L33) and [`#L57-L92`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/telegram/utils.ts#L57-L92) (`makeTelegramRequest`) do nothing but `JSON.stringify(body)` and `POST` it to `https://api.telegram.org/bot<token>/sendMessage`. No escaping/sanitization step exists anywhere in this call chain.

#### 3. How `payload.name` gets the malicious raw label with zero client-side gating: the gift/owner path, not the dead "favorite" path

The reachability argument in this report rests entirely on the **owner** branch, which resolves independently of any Manager frontend code:

[`workers/api-worker/src/queues/event-ingestion.ts#L75-L87`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L75-L87) (`collectRecipientsForEvent`):

```ts
export function collectRecipientsForEvent(
  event: ExpiryEvent,
  ownerToUserId: Map<string, string>,
  favoriteUsersByName: Map<string, Set<string>>,
): RecipientMap {
  const recipients: RecipientMap = new Map()

  if (event.owner) {
    const ownerUserId = ownerToUserId.get(event.owner.toLowerCase())
    if (ownerUserId) {
      recipients.set(ownerUserId, 'owned')
    }
  }
  ...
```

`ownerToUserId` is built purely from a raw address lookup — [`event-ingestion.ts#L146-L157`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L146-L157):

```ts
const owners = ownerAddresses.length > 0
  ? yield* intoDbResult(
      db.query.users.findMany({
        where: inArray(TABLE.users.address, ownerAddresses),
        columns: { id: true, address: true },
      }),
    )
  : []
```

and [`#L177-L179`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L177-L179):

```ts
const ownerToUserId = new Map(
  owners.map((owner) => [owner.address.toLowerCase(), owner.id]),
)
```

`ownerAddresses` itself comes from `event.owner` — [`#L124-L131`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L124-L131) — which is populated by the expiry-discovery indexer client purely from an on-chain ownership fact reported by the ENS subgraph, at [`workers/api-worker/src/services/expiry-discovery/indexer.ts#L43-L55`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/indexer.ts#L43-L55) (`name`/`owner.id` are bare `v.string()`, no length/character restriction) and [`indexer.ts#L135`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/indexer.ts#L135), and forwarded unmodified by [`workers/api-worker/src/services/expiry-discovery/index.ts#L35-L47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/index.ts#L35-L47) (`buildExpiryEvents`, `owner: domain.owner` at L44) and [`workers/api-worker/src/types/events/index.ts#L6-L13`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/types/events/index.ts#L6-L13) (`expiryEventSchema`, again bare `v.string()`). At no point does this path invoke `apps/manager`'s `parseName`/`normalize()` gates or any other client-side validator — it is a server-side, address-keyed lookup driven entirely by real on-chain ownership.

This matters because the *other* candidate entry point — a victim favoriting the malicious name through Manager's UI — is dead. [`apps/manager/src/features/register-v2/utils/name-parser.ts#L11`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L11) declares:

```ts
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
```

checked at [`name-parser.ts#L73-L75`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L73-L75) — this blocklist covers every Markdown metacharacter this payload needs (`` ` ``, `*`, `[`, `]`, `(`, `)`). Independently, [`apps/manager/src/features/profile/service/profileOwner.ts#L24`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileOwner.ts#L24) calls `normalizeEthName`, which wraps the real ENSIP-15 `normalize()` from `viem/ens` ([`apps/manager/src/features/profile/service/profileName.ts#L1,L14-L19`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileName.ts#L14-L19)) and returns `null` on the same malformed label. Both gates fire before the profile page (and its favorite button) can render, so a name containing the required breakout characters can never be favorited through any reachable Manager UI surface. The gift/owner path bypasses both gates entirely because it never runs any Manager frontend code — it is a raw DB address match fed by a raw on-chain ownership fact.

#### 4. `event.name` is copied verbatim into the stored notification payload

[`event-ingestion.ts#L223-L233`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L223-L233):

```ts
notificationsToInsert.push({
  user_id: userId,
  kind: 'name-expiry',
  payload: {
    name: event.name,
    expiryDate: event.expiryDate * 1000,
    isOwner: watchReason === 'owned',
    watchReason,
  },
  idempotency_key: idempotencyKey,
})
```

This row is inserted into Postgres and later read back verbatim by `deliverTelegramNotification` (step 2 above) and handed to the vulnerable template.

#### 5. Delivery is real-time-gated only by a real, shipped, opt-in toggle — one the app itself nudges on

[`workers/api-worker/src/queues/event-ingestion.ts#L54-L66`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L54-L66) (`shouldCreateExternalDeliveries`) gates the `owned` watch reason on `settings.owned_name_expiry` alone. The column defaults to `false` at the schema level ([`workers/api-worker/src/core/database/schema/notifications.ts#L143`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/database/schema/notifications.ts#L143)), but Manager's own post-registration onboarding screen sets it **on by default**: [`apps/manager/src/features/register-v2/workflow/registering/components/NotificationSettings.tsx#L23-L27`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/registering/components/NotificationSettings.tsx#L23-L27) calls `useNotificationPreferencesForm({ nameExpiryDefaultWhenUnset: true, ... })`, and [`apps/manager/src/features/notifications/settings/preferences.tsx#L55-L63`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/notifications/settings/preferences.tsx#L55-L63) uses that flag as the field's initial value:

```ts
ownedNameExpiry:
  preferences.data?.settings?.ownedNameExpiry ??
  nameExpiryDefaultWhenUnset,
```

The screen itself is titled *"Never lose your name to expiry."* ([`NotificationSettings.tsx#L33`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/registering/components/NotificationSettings.tsx#L33)) and is shown to every user immediately after they register or renew a name of their own — i.e., `owned_name_expiry: true` is the app's own encouraged, shipped default for any active Manager user who has added a Telegram contact method, entirely independent of ever encountering an attacker's gifted name.

#### 6. Telegram legacy Markdown (V1) parsing turns the injected label into live formatting and a live link

Per Telegram's Bot API documentation (`https://core.telegram.org/bots/api#formatting-options`, `#markdown-style`, fetched during this engagement): *"To escape characters '\_', '*', '`', '[' outside of an entity, prepend the character '\' before them"*; *"Escaping inside entities is not allowed, so entity must be closed first and reopened again"*; *"entities must not be nested"*. Unmatched entities make` sendMessage\` fail with a 400 error rather than silently neutralizing the payload — so the attacker must, and (holding full control of the entire label) trivially can, balance every opened primitive.

Given the label `poc-tg-mdinj-<ts>` + `` ` `` + `*🔒 ENS Security Alert:*` + free text + `[Verify your wallet now — official ENS](https://ens-domains-secure-verify.example/action)` + padding newlines + `` ` ``, substituted into the owner-branch template, the real production code built exactly this outbound `text` (captured verbatim in `evidence/04-captured-telegram-sendMessage-request.json` of the PoC):

```
⚠️ *Domain Expiration Alert*

Your domain `poc-tg-mdinj-1789154480`*🔒 ENS Security Alert:* Your wallet requires immediate re-verification

[Verify your wallet now — official ENS](https://ens-domains-secure-verify.example/action)





`.eth` will expire in *3 days*.

Don't forget to renew your domain!
```

* the template's opening backtick pairs with the label's **first** embedded backtick → `poc-tg-mdinj-<ts>` renders as an innocuous inline-code "domain chip";
* `*🔒 ENS Security Alert:*` is now outside any code span → renders **bold**, fully attacker-controlled fake "official" alert text;
* `[Verify your wallet now — official ENS](https://ens-domains-secure-verify.example/action)` renders as a genuine clickable link with attacker-chosen display text, independent of its destination;
* the label's own trailing backtick opens a fresh code span that closes cleanly against the template's own trailing backtick once `.eth` is appended — an unambiguous, well-formed message, not a malformed one that Telegram would reject.

I independently re-parsed this exact captured string against a from-scratch reference implementation of the documented legacy-Markdown entity rules (not part of the production call chain — corroboration only; `evidence/06-independent-markdown-v1-parse.json`). It resolves to the claimed entities: a bold `🔒 ENS Security Alert:` span, a `text_link` entity with text `Verify your wallet now — official ENS` and url `https://ens-domains-secure-verify.example/action`, and two small code spans — i.e., a victim reading this message in the real ENS bot chat sees:

```
⚠️ Domain Expiration Alert

Your domain poc-tg-mdinj-<ts> 🔒 ENS Security Alert: Your wallet requires immediate re-verification

Verify your wallet now — official ENS   [clickable link → attacker URL]




.eth will expire in 3 days.

Don't forget to renew your domain!
```

with the genuine trailing template text pushed below the fold by the padding blank lines.

#### 7. The same-payload button URL is malformed-looking but does NOT cause the message to be rejected — confirmed from Telegram's own published server source, not assumed

The template's `buttons` field (step 1) builds `url: https://app.ens.domains/${payload.name}` from the identical raw label, so the constructed button URL also contains backticks, asterisks, brackets, parentheses, spaces, newlines, and emoji. Telegram's real Bot API does reject some malformed inline-keyboard button URLs with a `400 BUTTON_URL_INVALID` error that fails the entire `sendMessage` call — so whether the whole notification (not just the button) is actually delivered turns on exactly what that validator checks. I determined this from Telegram's own published, MIT/Boost-licensed server-side source (`github.com/tdlib/td`, commit `d1085f9cebc5a62379991ae1652673954f229c1f`), which implements the Bot API server this program's real target depends on, rather than assuming either outcome:

* [`td/telegram/InlineKeyboardButton.cpp#L245-L261`](https://github.com/tdlib/td/blob/d1085f9cebc5a62379991ae1652673954f229c1f/td/telegram/InlineKeyboardButton.cpp#L245-L261) — for a plain `inlineKeyboardButtonTypeUrl` (exactly what this template builds — not a stricter `login_url`/`web_app` button), the button URL is validated by `LinkManager::check_link(url)` with no `https_only` override.
* [`td/telegram/LinkManager.cpp#L1926-L1993`](https://github.com/tdlib/td/blob/d1085f9cebc5a62379991ae1652673954f229c1f/td/telegram/LinkManager.cpp#L1926-L1993) (`check_link`/`check_link_impl`) — for an ordinary `http(s)://` URL, this calls `HttpUrl::parse_url` and then requires only that the parsed host contain a dot (`host_.find('.') != npos`).
* [`tdutils/td/utils/HttpUrl.cpp#L47-L193`](https://github.com/tdlib/td/blob/d1085f9cebc5a62379991ae1652673954f229c1f/tdutils/td/utils/HttpUrl.cpp#L47-L193) (`parse_url`) — the scheme (`http`/`https`) and the **host** are checked against a restrictive charset (`check_url_part`, L141-165), but everything **after** the host — the query/path, exactly where the attacker's entire label lands — goes through the loop at L122-140, which has **no error path**: any byte `<= 0x20` (control characters, including `\n`/`\r`, and the space character itself) is percent-encoded, and every other byte — including `` ` ``, `*`, `[`, `]`, `(`, `)`, and raw UTF-8 emoji bytes `>= 0x80` — passes through completely unvalidated.
* [`td/telegram/misc.cpp#L76-L161`](https://github.com/tdlib/td/blob/d1085f9cebc5a62379991ae1652673954f229c1f/td/telegram/misc.cpp#L76-L161) (`clean_input_string`, the remaining check `InlineKeyboardButton.cpp` applies to the result) fails only on malformed UTF-8; it otherwise just normalizes control characters and never rejects unusual-but-valid ones.

Because the button's host here is the fixed literal `app.ens.domains` (always a valid, dotted, alnum host) and the attacker's entire label lives in the path, this validation chain **accepts** the constructed button URL — Telegram's real server would silently percent-encode the control bytes/spaces in the path server-side and deliver the message, not reject it with `BUTTON_URL_INVALID`. I ported this exact decision logic (cited line-for-line above) into the PoC as an independent check (`evidence/07-button-url-tdlib-validation.json`) and additionally wired the PoC's Telegram-API stub to actually run it against every constructed button URL before deciding whether to simulate acceptance or a real `400 BUTTON_URL_INVALID` response — for this payload it returns **ACCEPTED**, and the PoC's own real production `sendMessage` call receives a success response exactly as the real API would give it, confirmed rather than assumed:

```json
{
  "verdict": "ACCEPTED",
  "rawButtonUrl": "https://app.ens.domains/poc-tg-mdinj-1789154480`*🔒 ENS Security Alert:* Your wallet requires immediate re-verification\n\n[Verify your wallet now — official ENS](https://ens-domains-secure-verify.example/action)\n\n\n\n\n\n`.eth",
  "normalizedUrl": "https://app.ens.domains/poc-tg-mdinj-1789154480`*🔒%20ENS%20Security%20Alert:*%20Your%20wallet%20requires%20immediate%20re-verification%0A%0A[Verify%20your%20wallet%20now%20—%20official%20ENS](https://ens-domains-secure-verify.example/action)%0A%0A%0A%0A%0A%0A`.eth"
}
```

The button's own destination becomes a garbled, non-functional URL as a side effect (this was already noted as a low-impact secondary detail in the original triage, not a separate claim this report relies on) — but the **message itself, including the primary text-injection payload, is delivered**. This confirms the report's central claim ("the victim receives the message") holds under Telegram's real, documented validation behavior, not merely under this PoC's own assumptions.

## Root Cause

The root cause is entirely first-party: `templates/telegram.ts` is code this program owns and authored, and it is the **only** place in the entire call chain that ever builds the outbound `text` string. Telegram is not misbehaving — it faithfully parses exactly the Markdown it is documented and expected to parse; the defect is that `workers/api-worker` hands it attacker-controlled, unescaped text inside a markup-interpreted field.

**First-party vs. third-party adjudication.** The only out-of-scope actions in this chain are (a) the attacker's label registration directly against the ENS registrar contracts, and (b) the ENS subgraph's report of the resulting on-chain ownership. Both are **data-poisoning preconditions that happen first**, not a later call an attacker or victim must make to *complete* the harm — every step that actually constructs and sends the malicious message (`event-ingestion.ts` → `deliverTelegramNotification` → `makeTelegramRequest`) is 100% first-party `workers/api-worker` code, running automatically, with no further action or call into any out-of-scope component required by anyone. This is the identical shape as the program's own already-confirmed "Portal V1 history trusts unauthenticated matching-log rows" root cause (an attacker-controlled on-chain action plus an out-of-scope indexer read, completed entirely by in-scope code that fails to validate before treating the result as trusted) — and, more directly, the *gift-to-owner* precondition itself ("register a name on-chain, then transfer it to a victim's wallet, bypassing all Manager-side client validation") is the **exact same precondition shape** as this program's four already-confirmed "gifted name" Manager-dashboard bugs (the CBOR/ABI decoder hang, the ZEN/coin-121 decoder hang, the primary-contacts quadratic dedup hang, and the unbounded profile-links rendering hang — all triggered by an attacker gifting a crafted name directly to a victim's address with zero interaction). Those four confirmed roots establish that this program already treats "attacker registers on-chain, then gifts the name to a victim, then in-scope first-party code processes it without adequate validation" as a legitimate, in-scope attack shape — not a scope violation. This finding is that exact shape, with the vulnerable first-party code living in `workers/api-worker` instead of `apps/manager`, and the "adequate validation" gap being unescaped Markdown interpolation instead of unbounded decoding work. `workers/api-worker` is listed in the program's scope on equal footing with `apps/manager`/`apps/portal`, and the only modification/composition this report relies on is entirely inside it.

## Impact Explanation

This meets the High bucket's "persistent static-content injection without JS" element by element:

* **Unauthorized:** the attacker is never the victim's Manager account holder, never authenticates as the victim, and never needs the victim's cooperation. The victim's only action (adding a verified Telegram channel and leaving the app's own defaulted-on `owned_name_expiry` toggle enabled) is an ordinary, unrelated, earlier step.
* **Persistent:** the malicious content is not a one-off reflection. It is generated deterministically by the hourly `runExpiryDiscoveryCron` for as long as the name remains in the victim's on-chain custody and unexpired, and the underlying `notifications` row is durably stored in Postgres.
* **Without JS:** the attack is pure abuse of a text/markup primitive (Markdown V1 entity parsing), not script injection or code execution.
* **Replaces/augments trusted content with attacker-chosen content:** the real `⚠️ *Domain Expiration Alert*` header (outside attacker control) frames attacker-chosen bold "official" text and a deceptive clickable link, delivered inside the same bot conversation the victim already trusts for legitimate ENS alerts — functionally the bot-message equivalent of injecting persistent content into a page the victim trusts. This delivery is confirmed, not assumed: Finding Description step 7 shows from Telegram's own published server source that the malformed-looking button URL sharing the same payload does not cause the whole message to be rejected.

I evaluated and rejected Critical (no wrong-address/tx-detail in a connected-wallet flow, no NFT/fund theft, no code execution) and Medium (this is not reflected — it is deterministic, persistent, and reaches the victim automatically once, not on every visit like an open redirect, but through a durable stored row and a scheduled job, materially worse than a single-shot redirect).

## Likelihood Explanation

The exploit requires only ordinary, unprivileged capabilities:

1. register any available ENS label on-chain, choosing its raw bytes freely (no ENSIP-15 enforcement at the registrar level);
2. call the standard ERC-1155 `safeTransferFrom` to the victim's address (no recipient consent required for an EOA); and
3. wait for the program's own hourly expiry-discovery cron to run.

The victim needs no wallet connection, no click, and no awareness that anything happened — the only precondition on the victim's side is a Manager account with a verified Telegram channel and the shipped, commonly-defaulted-on `owned_name_expiry` setting, both ordinary product usage. The attack does not depend on admin, governance, oracle, indexer-operator, or third-party credential compromise, and it is fully deterministic once indexed — every affected victim reaches the same forged message.

## Attack Path / Reproduction Path

1. Victim `V` has (or later obtains) a Manager account with a verified Telegram channel and `owned_name_expiry` enabled (the app's own post-registration screen defaults this on).
2. Attacker `A` registers a raw ENS label containing Markdown-V1 breakout characters directly against the real deployed V2 `ETHRegistrar` — no on-chain ENSIP-15 gate blocks it.
3. `A` calls `safeTransferFrom(A, V, tokenId, 1, "0x")` on the real deployed V2 ETH Registry, gifting the name to `V`. `V` performs no action and needs no awareness of this transfer.
4. On the next hourly run, `workers/api-worker`'s `runExpiryDiscoveryCron` asks the ENS indexer for names expiring inside its lookahead window and receives the gifted name with `owner = V`.
5. `handleEventIngestionQueue` resolves `V`'s Manager account via a raw address lookup (`collectRecipientsForEvent`, `ownerToUserId`) — no Manager frontend code, no character validator, is ever invoked.
6. Because `V`'s `owned_name_expiry` setting is `true`, a `notifications` row is inserted with `payload.name = A`'s raw crafted label + `.eth`, and a `telegram` delivery job is enqueued.
7. `handleTelegramQueue` → `deliverTelegramNotification` calls `telegramTemplates['name-expiry'](payload)`, producing an unescaped Markdown string and a same-payload button URL, and `makeTelegramRequest` POSTs it to the real Telegram Bot API with `parse_mode: 'Markdown'`.
8. Telegram's real server validates the request; per Finding Description step 7, the button URL's malformed-looking path does not cause rejection, so the message is delivered to `V`'s real chat with the real ENS bot. `V` sees the genuine `⚠️ *Domain Expiration Alert*` header plus attacker-chosen bold text and a clickable phishing link with attacker-chosen display text — with zero interaction beyond having previously enabled a shipped notification setting.

## External Preconditions

* The attacker can register any available ENS label on-chain and pay the ordinary registration fee (no special privilege).
* The attacker can execute one ordinary ERC-1155 `safeTransferFrom` call to the victim's address (no victim consent required for an EOA recipient).
* The victim already has a Manager account with a verified Telegram channel.
* The victim's `owned_name_expiry` notification setting is enabled — a real, shipped, independently toggleable setting that Manager's own post-registration onboarding screen defaults to `true`.

## Internal Preconditions

* `templates/telegram.ts`'s `name-expiry` template continues to interpolate `payload.name` unescaped inside a `parse_mode: 'Markdown'` field and inside the button `url` field.
* `event-ingestion.ts`'s owner-recipient resolution continues to key off a raw on-chain-address-to-account lookup, independent of any Manager frontend name validator.
* No component between the stored `notifications.payload` row and the outbound `sendMessage` call performs escaping or Markdown-entity sanitization.

## Affected Code

All links reference frozen competition commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

* [`workers/api-worker/src/services/delivery/templates/telegram.ts#L28-L49`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/delivery/templates/telegram.ts#L28-L49) — the vulnerable sink: `payload.name` interpolated raw into a `parse_mode: 'Markdown'` string and into the button `url`.
* [`workers/api-worker/src/services/delivery/telegram.ts#L23-L92`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/delivery/telegram.ts#L23-L92) — reads the stored payload, calls the template, forwards the result unescaped to the real Telegram API call.
* [`workers/api-worker/src/services/telegram/utils.ts#L33`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/telegram/utils.ts#L33) and [`#L57-L92`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/telegram/utils.ts#L57-L92) — `makeTelegramRequest`; no escaping/sanitization anywhere in this function.
* [`workers/api-worker/src/queues/telegram.ts#L9-L58`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/telegram.ts#L9-L58) — the Telegram delivery queue consumer, no sanitization layer added.
* [`workers/api-worker/src/queues/event-ingestion.ts#L54-L66`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L54-L66) — `shouldCreateExternalDeliveries`, the `owned_name_expiry` gate.
* [`workers/api-worker/src/queues/event-ingestion.ts#L75-L87`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L75-L87) — `collectRecipientsForEvent`'s owner branch, the recipient-resolution path that bypasses all Manager frontend validation.
* [`workers/api-worker/src/queues/event-ingestion.ts#L146-L157`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L146-L157) and [`#L177-L179`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L177-L179) — the raw address-to-account lookup.
* [`workers/api-worker/src/queues/event-ingestion.ts#L223-L233`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L223-L233) — copies `event.name` verbatim, unescaped, into the stored notification payload.
* [`workers/api-worker/src/services/expiry-discovery/indexer.ts#L43-L55`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/indexer.ts#L43-L55) and [`#L135`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/indexer.ts#L135) — no length/character restriction on the indexed `name`/`owner`.
* [`workers/api-worker/src/services/expiry-discovery/index.ts#L35-L47`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/index.ts#L35-L47) — `buildExpiryEvents`, forwards `owner`/`name` unmodified.
* [`workers/api-worker/src/types/events/index.ts#L6-L13`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/types/events/index.ts#L6-L13) — `expiryEventSchema`, unrestricted `name`/`owner` strings at the queue-message boundary.
* [`workers/api-worker/src/core/database/schema/notifications.ts#L138-L150`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/core/database/schema/notifications.ts#L138-L150) — `userNotificationSettings`, the `owned_name_expiry` column.
* [`apps/manager/src/features/register-v2/workflow/registering/components/NotificationSettings.tsx#L23-L27`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/registering/components/NotificationSettings.tsx#L23-L27) and [`apps/manager/src/features/notifications/settings/preferences.tsx#L55-L63`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/notifications/settings/preferences.tsx#L55-L63) — corroborating: the app's own registration-flow onboarding defaults `owned_name_expiry` on.

## Recommended Mitigation

Never interpolate untrusted ENS-name text directly into a `parse_mode`-sensitive field. Either:

1. Escape every legacy-Markdown metacharacter (`` ` ``, `*`, `_`, `[`) in `payload.name` (and any other untrusted field reaching a Telegram template) with a backslash before interpolation, matching Telegram's own documented escaping rule; or, preferably,
2. Switch to `parse_mode: 'MarkdownV2'` (or no `parse_mode` at all for fields carrying untrusted text) and construct the message using Telegram's explicit `entities` array for the fixed, trusted formatting (the header, "days left", the renew button), rather than string concatenation, so untrusted text can never be interpreted as markup regardless of its content.

Also stop building the button's `url` field from raw `payload.name`. Confirmed in Finding Description step 7, Telegram's real server accepts this malformed-looking URL rather than rejecting the whole message, so it is not currently a delivery-blocking issue — but it is still a broken, non-functional "Renew Now" destination today, and a defense-in-depth fix (URL-encode the label with `encodeURIComponent`, or link to a fixed lookup route instead of embedding the raw name in the path) removes the risk entirely rather than relying on the specific leniency of Telegram's current validator.

Apply the same escaping treatment to the `name-transferred` template's `payload.name` interpolation (`templates/telegram.ts` — currently unreachable in production, but the same root cause). Add regression coverage asserting that a label containing every legacy-Markdown metacharacter round-trips as literal text in the constructed `sendMessage` body, and that the button URL is well-formed after encoding.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/b426a1b811f42477be56882f76693ead/91019c543d0a9b211b1f8538a102f664d10436d1>

## Proof of Concept

**Description**

This is a real, non-mocked, end-to-end proof. It runs the frozen `workers/api-worker` production source against a local Anvil fork of Sepolia carrying the real deployed ENS V2 contracts, and against the project's own local Postgres + `local-neon-http-proxy` dev-database stack (the real `drizzle-orm/neon-http` driver path). Nothing in the vulnerable business-logic call chain is mocked or reimplemented:

* On-chain registration of the malicious label via the real deployed V2 `ETHRegistrar.commit()`/`register()`, and the real ERC-1155 `safeTransferFrom` gift-transfer on the real deployed V2 ETH Registry.
* `runExpiryDiscoveryCron` → `fetchExpiringNamesPage`/`executeIndexerQuery` (real `graphql-request` call, real response-schema validation) → `handleEventIngestionQueue` → `processExpiryEvents` (real Postgres reads/writes through the real production driver) → `handleTelegramQueue` → `deliverTelegramNotification` → `telegramTemplates['name-expiry']` (the vulnerable code) → `makeTelegramRequest` (real URL/body construction, real response-schema parsing).

The only stubs are infrastructure this program does not own and that is not the vulnerable code path: a local HTTP server standing in for the out-of-scope ENS subgraph (mirroring the project's own `e2e/helpers/mock-indexer.ts` pattern), an in-memory Cloudflare KV/Queue-transport stand-in that synchronously invokes the real consumer functions (exactly mirroring how a Cloudflare Queue trigger invokes its consumer), and the actual outbound `fetch()` to `https://api.telegram.org` (no real bot credentials exist for this engagement — the one exception the task explicitly authorizes). That stub does not blindly accept whatever is POSTed: it runs a hand-ported, line-cited reproduction of Telegram's own published button-URL validation logic (`github.com/tdlib/td`) against the real constructed request and only synthesizes a success response when that logic determines the real API would also accept it (see Finding Description step 7). Everything upstream of the transport call, including the exact JSON body construction, is the real production function.

The PoC (1) confirms on-chain registration accepts the malicious label, (2) transfers the name to a fresh victim address, (3) seeds the real database with the victim's `owned_name_expiry: true` account state, (4) drives the real production pipeline end to end, (5) captures and asserts on the exact resulting Telegram `sendMessage` payload, (6) independently re-parses the captured text against a from-scratch reference implementation of Telegram's documented legacy-Markdown entity rules, confirming the claimed bold/link entities, and (7) independently re-validates the captured button URL against Telegram's own published server-side URL-validation source, confirming it would be accepted, not rejected.

**Where to place files and their names**

Place the complete five-file PoC under the frozen repository root as follows:

```
poc/api-worker-telegram-markdown-injection/
├── .gitignore
├── README.md
├── SOURCE_SHA256SUMS
├── poc.ts
├── run.sh
└── evidence/
    ├── 00-full-run.log
    ├── 01-onchain.json
    ├── 03-pipeline-run-summary.json
    ├── 04-captured-telegram-sendMessage-request.json
    ├── 05-db-state-after-run.json
    ├── 06-independent-markdown-v1-parse.json
    └── 07-button-url-tdlib-validation.json
```

**Tool and dependency versions**

```
Node.js:                       v24.18.0 (runner accepts Node.js 20+)
pnpm:                           10.27.0
tsx:                            4.21.0
Docker Compose:                v2
Anvil (foundry) image digest:  sha256:0c00cb0bda1ab1b91c9a6bf60f4c76c09c1a8870824b6d4718afbabacf6f9a17
PostgreSQL base image digest:  sha256:67f41722b7a8cbdb868a44a4995c846eddfdc2973bccb291ce937dce88ad5675 (postgres:17, workers/api-worker/Dockerfile)
local-neon-http-proxy digest:  sha256:cd2ae14edf2feafbc3330492de5c80506f77274c3bd013154cdef697bdeb768a
```

**PoC Link**

<https://gist.github.com/aydarkhusaenov/b426a1b811f42477be56882f76693ead/91019c543d0a9b211b1f8538a102f664d10436d1>

**Clean clone, checkout, and installation**

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/b426a1b811f42477be56882f76693ead.git \
  poc/api-worker-telegram-markdown-injection
git -C poc/api-worker-telegram-markdown-injection checkout --detach \
  91019c543d0a9b211b1f8538a102f664d10436d1

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
```

**Exact run command**

Run from the frozen repository root:

```bash
bash poc/api-worker-telegram-markdown-injection/run.sh
```

The runner verifies the pinned commit and package hashes, starts a local Postgres + `local-neon-http-proxy` (the project's own `workers/api-worker/compose.yaml`) and applies the project's own drizzle migrations, starts a digest-pinned local Anvil fork of Sepolia and verifies the exact fork block hash, runs the PoC, saves full evidence to `evidence/`, and tears everything down on exit. Optional overrides: `POC_ANVIL_PORT`, `POC_INDEXER_PORT`, `SEPOLIA_FORK_RPC`, `POC_LOG_DIR`.

**Fresh logs proving the bug**

A fresh clean-clone run produced (excerpted; full transcript in `evidence/00-full-run.log`):

```
=== PHASE A: on-chain registration + gift-transfer ===
[registrar] isAvailable(maliciousLabel) = true
[registrar] REAL on-chain register() succeeded. tx: 0x18b9f8d7dd43dce08fdee4b62fdae5ec04bb5d8a4a7ed48f37279cd0c92e6e1e status: success
[registry] REAL safeTransferFrom(attacker -> victim) succeeded. tx: 0xd6ebe988aee3362f263a4e88b00913e430e284efb1b4dd85f2b6b01d3e5254ed
[registry] verified final ERC-1155 balances: attacker=0 victim=1

=== PHASE B: real production event-ingestion -> Telegram delivery pipeline ===
[db] real INSERT into users: { id: '01a091eb-4be1-7445-b489-774e4aabc580', address: '0xcefd5833461765c66fb62b66ce9a322866aabaa1' }
[db] real INSERT into user_channels: { channel: 'telegram', target: '918273645', status: 'verified' }
[db] real INSERT into user_notification_settings: owned_name_expiry=true
[cron] calling REAL runExpiryDiscoveryCron(env) ...
{"level":"debug","message":"Indexer query succeeded","stageId":"7d","domainCount":1,...}
{"level":"debug","message":"Telegram notification delivered","jobId":"01a091eb-4dab-7000-8000-039f21b44983","messageId":900000001}
[cron] runExpiryDiscoveryCron completed: { totalEnqueued: 1, failedStages: 0 }
[assert] all pipeline + payload assertions PASSED
[assert] real DB state assertions PASSED: {
  notification: { kind: 'name-expiry', payload: { isOwner: true, watchReason: 'owned', ... } },
  delivery: { status: 'delivered', provider_msg_id: '900000001' }
}
[evidence] wrote 04-captured-telegram-sendMessage-request.json

=== PHASE C: independent corroboration against Telegram's documented Markdown V1 rules ===
[markdown-check] bold entities: [ 'Domain Expiration Alert', '🔒 ENS Security Alert:', '3 days' ]
[markdown-check] text_link entity: { text: 'Verify your wallet now — official ENS', url: 'https://ens-domains-secure-verify.example/action' }

=== PHASE D: is the malformed-looking button URL actually ACCEPTED by Telegram's real API? ===
[button-url] raw url built by the real production template: "https://app.ens.domains/poc-tg-mdinj-1789154480`*🔒 ENS Security Alert:* Your wallet requires immediate re-verification\n\n[Verify your wallet now — official ENS](https://ens-domains-secure-verify.example/action)\n\n\n\n\n\n`.eth"
[button-url] tdlib-style check_link() verdict: ACCEPTED
[button-url] server-side normalized url (percent-encoding bytes <= 0x20 only): "https://app.ens.domains/poc-tg-mdinj-1789154480`*🔒%20ENS%20Security%20Alert:*%20Your%20wallet%20requires%20immediate%20re-verification%0A%0A[Verify%20your%20wallet%20now%20—%20official%20ENS](https://ens-domains-secure-verify.example/action)%0A%0A%0A%0A%0A%0A`.eth"
[evidence] wrote 07-button-url-tdlib-validation.json

POC_MARKER {"marker":"ENS_R1638_TELEGRAM_MARKDOWN_INJECTION_PASS", ...}

PoC PASSED. Evidence: poc/api-worker-telegram-markdown-injection/evidence
```

The exact captured outbound payload (`evidence/04-captured-telegram-sendMessage-request.json`):

```json
{
  "chat_id": "918273645",
  "parse_mode": "Markdown",
  "text": "⚠️ *Domain Expiration Alert*\n\nYour domain `poc-tg-mdinj-1789154480`*🔒 ENS Security Alert:* Your wallet requires immediate re-verification\n\n[Verify your wallet now — official ENS](https://ens-domains-secure-verify.example/action)\n\n\n\n\n\n`.eth` will expire in *3 days*.\n\nDon't forget to renew your domain!",
  "reply_markup": {
    "inline_keyboard": [[{ "text": "🔄 Renew Now", "url": "https://app.ens.domains/poc-tg-mdinj-1789154480`*🔒 ENS Security Alert:* Your wallet requires immediate re-verification\n\n[Verify your wallet now — official ENS](https://ens-domains-secure-verify.example/action)\n\n\n\n\n\n`.eth" }]]
  }
}
```

and the real database rows the real pipeline wrote (`evidence/05-db-state-after-run.json`): a `notifications` row with `kind: 'name-expiry'`, `payload.isOwner: true`, `payload.watchReason: 'owned'`, and `payload.name` equal to the attacker's raw crafted label plus `.eth`; and a `notification_deliveries` row with `channel: 'telegram'`, `status: 'delivered'`, and `provider_msg_id: '900000001'`.
