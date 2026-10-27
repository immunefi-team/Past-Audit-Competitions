# 92878 w a insight persistent markdown content injection into telegram expiry notification messages via an unsanitized ens name label

**Submitted on Sep 13th 2026 at 01:35:19 UTC by @deltasec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92878
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

The notification worker builds Telegram expiry-alert messages by interpolating an ENS name's on-chain `label` directly into a `parse_mode: 'Markdown'` message string, with no escaping and no normalization at any point in the pipeline. Because ENS labels are arbitrary attacker-chosen strings on-chain (nothing at the contract or indexer layer restricts their character set), an attacker can register a name whose label is itself a crafted Markdown payload, transfer the resulting NFT to a victim with a single no-consent-required ERC-721 transfer, and have the official ENS notification bot deliver a message to the victim that renders an attacker-controlled clickable hyperlink inside an otherwise-legitimate "⚠️ Domain Expiration Alert."

## Vulnerability Details

**Untrusted source.** The `.eth` registrar and the subgraph/indexer both store and return a name's label verbatim. There is no contract-level or indexer-level restriction on which characters a label may contain — normalization (ENSIP-15) is only ever enforced by client-side UI, never on-chain or in the indexer. `payload.name` therefore can carry arbitrary Markdown metacharacters (`` ` ``, `[`, `]`, `(`, `)`, `*`, `_`) end-to-end.

**Vulnerable pipeline** (every hop trusts the label as-is; zero normalization/escaping anywhere):

1. `workers/api-worker/src/services/expiry-discovery/indexer.ts:134` — fetches `domain.name` verbatim from the subgraph/indexer.
2. `workers/api-worker/src/services/expiry-discovery/index.ts:35-47` (`buildExpiryEvents`) — copies `domain.name` straight into a `name_expiring` event's `name` field.
3. `workers/api-worker/src/types/events/index.ts` — `expiryEventSchema` validates `name: v.string()` only; no charset or length restriction.
4. `workers/api-worker/src/queues/event-ingestion.ts:~226` — persists `notifications.payload.name = event.name` unchanged.
5. **Sink** — `workers/api-worker/src/services/delivery/templates/telegram.ts:28-49` (`telegramTemplates['name-expiry']`):

   ```ts
   const ownerText = payload.isOwner
     ? `Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\n...`
     : `The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.`
   return { text: `⚠️ *Domain Expiration Alert*\n\n${ownerText}`, parseMode: 'Markdown', ... }
   ```

   `workers/api-worker/src/services/delivery/telegram.ts:68` sends this `text` to the Telegram Bot API `sendMessage` with `parse_mode: message.parseMode` — no escaping is applied anywhere between step 1 and this call.

Confirmed-safe sinks (checked, not affected):

* Push notifications render as plaintext (Telegram/APNs Markdown parsing does not apply to push payloads).
* The in-app manager inbox renders `name` as a React text node (auto-escaped).

Related, unverified sink noted for completeness: `workers/api-worker/src/services/delivery/email.ts` passes `dynamicData.name = payload.name` into a SendGrid dynamic template referenced only by ID (`env.SENDGRID_TEMPLATE_IDS`); the template body itself is not in this repository. If that dashboard-hosted template uses a raw/triple-brace field for `name` instead of an HTML-escaped one, the same label would inject arbitrary HTML into an email. This cannot be confirmed from the codebase and is not part of the impact claimed below — it is listed under Recommendation as a hardening item.

**Corroboration this is an omission, not an intentional design choice.** The application already treats this character class as dangerous at one specific entry point: `apps/manager/src/features/register-v2/utils/name-parser.ts` blocklists ``/[&*@#$%^()[\]{}|\\:;"'<>?,=+~\`!]/`` on names typed into the registration UI. That guard only covers input typed into this one form. Labels reaching the notification pipeline come from the indexer and reflect whatever was registered on-chain by any method — including a direct `ETHRegistrar` contract call that never touches this UI — and are trusted verbatim from that point forward. The fix belongs on the read/deliver path, which currently has no equivalent guard at all.

## Impact Details

This is a **persistent** (the label lives on-chain), **cross-user** (the attacker never touches the victim's session, keys, or credentials) content-injection primitive against an official, trusted notification channel. It matches the audit's High-severity impact: *"Injecting/modifying static content without JavaScript (persistent HTML injection, text replacement, arbitrary file uploads)."* Delivered via Telegram, the attacker-chosen `[text](url)` link renders as a live, clickable hyperlink inside an authentic ENS-bot conversation thread and an otherwise-legitimate expiration-alert message — a high-trust setup for phishing (fake renewal page, wallet-drainer link, credential harvesting). The inline keyboard "renew" button's `url` is built from the same unescaped `payload.name`, giving a second, lower-value injection surface (Telegram validates button `url` fields more strictly than message text).

No victim interaction is required to *deliver* the message when the victim already owns the name (see Proof of Concept, path A); one victim action (favoriting the attacker's name) is required for the alternate delivery path (path B). Either way, the attacker never interacts with the victim's wallet, session, or app account.

### Risk Breakdown

* **Difficulty to exploit:** Low. Registering a label with the required characters is a standard, permissionless `ETHRegistrar` call; delivering it to a victim who owns it is a single standard ERC-721 `safeTransferFrom`.
* **Likelihood:** Medium. Requires the victim to be a registered app user with a verified Telegram channel and expiry notifications enabled — a normal, encouraged app configuration, not an edge case.
* **Impact:** High (per audit's impact table) — persistent, cross-user content injection into an authenticated notification channel, enabling convincing phishing against ENS users via a channel they trust.

## Recommendation

1. **Normalize at ingestion.** Run every name pulled from the indexer through ENSIP-15 normalization before it is stored in `notifications.payload` or enqueued as an event; reject or strip names that fail normalization rather than trusting the indexer's raw string.
2. **Escape at every delivery sink, defense in depth, regardless of (1):**
   * Telegram: switch to `MarkdownV2` and escape all of ``_*[]()~`>#+-=|{}.!`` in interpolated fields per Telegram's documented escaping rules, or drop Markdown parsing for interpolated segments entirely (plain text body + separate button entities for links).
   * SendGrid: confirm — and if necessary change — the dashboard-hosted dynamic template to use `{{name}}` (HTML-escaped) rather than `{{{name}}}` (raw), for the same `payload.name` field.

## References

* `workers/api-worker/src/services/expiry-discovery/indexer.ts:134`
* `workers/api-worker/src/services/expiry-discovery/index.ts:35-47`
* `workers/api-worker/src/types/events/index.ts`
* `workers/api-worker/src/queues/event-ingestion.ts:~226`
* `workers/api-worker/src/services/delivery/templates/telegram.ts:28-49`
* `workers/api-worker/src/services/delivery/telegram.ts:68`
* `workers/api-worker/src/services/delivery/email.ts` (related, unverified sink)
* `apps/manager/src/features/register-v2/utils/name-parser.ts` (corroborating, narrower existing guard)

## Proof of Concept

## Proof of Concept

1. Attacker registers label ``a`[Click to renew instantly →](https://ens-renew-portal.evil)`b`` directly via `ETHRegistrar` (balanced backticks, so it survives inside the template's own `` `${name}` `` wrapper) → full name `a\`\[...]\`b.eth\`.
2. Attacker calls `safeTransferFrom(attacker, victim, tokenId)` — plain ERC-721, no victim signature or consent needed. (Alternatively: victim favorites the name themselves — same delivery path, no transfer required.)
3. `expiry-discovery` indexes the name with `owner = victim`; `event-ingestion.ts` stores `payload.name` verbatim.
4. Delivery worker renders the Telegram template below and sends it to the victim's verified Telegram channel with `parse_mode: 'Markdown'`.

Step 4 reproduced standalone (verbatim copy of the production template function).

```js
// verbatim copy of workers/api-worker/src/services/delivery/templates/telegram.ts:28-49
function nameExpiryTelegramTemplate(payload) {
  const daysLeft = Math.ceil((payload.expiryDate - Date.now()) / (1000 * 60 * 60 * 24))
  const ownerText = payload.isOwner
    ? `Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\nDon't forget to renew your domain!`
    : `The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.`
  return {
    text: `⚠️ *Domain Expiration Alert*\n\n${ownerText}`,
    parseMode: 'Markdown',
    buttons: [[{ text: '🔄 Renew Now', url: `https://app.ens.domains/${payload.name}` }]],
  }
}

const maliciousName = 'a`[Click to renew instantly →](https://ens-renew-portal.evil)`b.eth'
// payload.name as stored verbatim by event-ingestion.ts from the indexer's domain.name
const message = nameExpiryTelegramTemplate({ name: maliciousName, expiryDate: Date.now() + 5 * 864e5, isOwner: true })
console.log(message.text)
```

Output — this is the literal `text` sent to Telegram's `sendMessage` with `parse_mode: 'Markdown'`:

```
⚠️ *Domain Expiration Alert*

Your domain `a`[Click to renew instantly →](https://ens-renew-portal.evil)`b.eth` will expire in *5 days*.

Don't forget to renew your domain!
```

Telegram parses this as valid legacy Markdown (the payload's own backticks balance the template's), rendering `[Click to renew instantly →](https://ens-renew-portal.evil)` as a live hyperlink inside an authentic-looking ENS bot alert. An unbalanced payload would instead cause `sendMessage` to fail outright — not render broken — which is why the payload above is deliberately balanced.
