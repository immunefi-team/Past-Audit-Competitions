# 89513 w a insight missing markdown escaping in notification worker allows attacker author ens branded telegram message content or silently suppress a victim s expiry warnings

**Submitted on Aug 19th 2026 at 17:23:42 UTC by @roulette6041 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89513
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (reflected), such as:
* Reflected HTML Injection
* Loading external site data

## Description

## Brief/Intro

The notification worker builds Telegram messages with `parse_mode: 'Markdown'` and interpolates the ENS name into the message body with no escaping anywhere in the delivery layer. ENS labels are attacker-chosen strings and the v2 contracts do not constrain their characters, so a backtick in the label closes the code span the template opened and the attacker controls the rest of the message. The victim receives a message from the genuine ENS bot carrying text the attacker wrote, including a tappable mention pointing at an account the attacker names, with the backticks consumed as delimiters so nothing looks malformed. A one-character variant of the same defect leaves the Markdown entities unbalanced instead, Telegram rejects the send, and the victim silently never receives the expiry warning they opted into.

### Risk Breakdown

* **Affected component:** `workers/api-worker`, notification delivery path.
* **Pre-conditions:** the victim has a verified Telegram channel ([`event-ingestion.ts:285-287`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L285-L287)); expiry notifications enabled, which is opt-in via `watchBasedNameExpiry`; the name is inside an expiry threshold of 30, 7 or 1 days; and the victim's address already exists in `users`, meaning they have logged in at least once.
* **Who can exploit:** any unprivileged party who can register a `.eth` name and transfer it. No role, no approval, and no interaction between attacker and victim.
* **Funds at risk:** none directly. The harm is a phishing channel with borrowed authenticity, plus suppression of a warning the user relies on to avoid losing a name.

## Vulnerability Details

### 1. The template interpolates the name into Markdown with no escaping

[`services/delivery/templates/telegram.ts:28-49`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/delivery/templates/telegram.ts#L28-L49) builds the `name-expiry` message. The name sits inside a backtick code span at `:34-35`, `parseMode` is set to `'Markdown'` at `:39`, and the inline-keyboard button URL concatenates the same value at `:44`.

Note that `'Markdown'` is Telegram's **legacy** parse mode, not `MarkdownV2`. In legacy Markdown a code span ends at the next backtick.

[`services/delivery/telegram.ts:68-73`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/delivery/telegram.ts#L68-L73) passes `message.text` and `message.parseMode` straight to the API, and [`services/telegram/utils.ts:56-70`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/telegram/utils.ts#L56-L70) does no processing on the way out.

### 2. The name reaches the template raw, verified on-chain

**The label is registerable.** Two names were registered on live Sepolia through the v2 registrar. Name A used the label `re` + backtick + `new`, Name B used the payload label. Both returned `status 0x1`.

**The backtick survives on chain.** The label registry at `0x532cd0cc4ac0793d838f71a67d29b2d790d18777` emits the label as a string. Decoding Name A's log gives length 6 and bytes `72 65 60 6e 65 77`, where `0x60` is the backtick. Nothing normalizes at write time.

**The indexer returns it verbatim:**

```json
{"name":"re`new.eth","labelName":"re`new","normalizedName":null,
 "isNormalized":false,"owner":{"id":"0xa3bfe86e5d412bbf14461ea5323f375a7c42f654"}}
```

**The worker requests only the raw field.** [`services/expiry-discovery/indexer.ts:13-27`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/expiry-discovery/indexer.ts#L13-L27) selects `name`, `expiryDate` and `owner { id }`. It never requests `normalizedName` and never requests `isNormalized`, so it cannot branch on normalization even in principle.

**The endpoint applies no normalization filter, in the worker's own query shape.** I ran the worker's shape against `https://graphql.ens.dev/` with no additional filter:

```
domains(where: { expiry_gt: 1816602980, expiry_lte: 1816602982 }, orderBy: expiryDate)
  -> {"name":"leo.canonicalId:3562705...","isNormalized":false,"normalizedName":null}
```

An unnormalized row comes back. Three separate expiry windows were tested and all three returned their unnormalized row.

**The deployed worker is pinned to that endpoint by configuration, not by a code default.** [`wrangler.jsonc:99`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/wrangler.jsonc#L99) sets `"ENS_INDEXER_GRAPHQL_URL": "https://graphql.ens.dev/"` in `vars`. This matters because an environment variable could otherwise override the code default.

### 3. Delivery needs no consent from the victim

[`queues/event-ingestion.ts:149-155`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/queues/event-ingestion.ts#L149-L155) resolves recipients by matching the indexer's `owner.id` against `users.address`, tagging them `watchReason: 'owned'`. ENS names are transferable tokens, and a transfer needs no approval from the recipient. So an attacker registers the crafted name and transfers it to the victim, and the victim becomes the owner the indexer reports.

### 4. Telegram parses it exactly as the mechanism predicts

I ran five probes against a throwaway bot. Two of them exist purely to remove an ambiguity, and that is the part worth reading.

| Probe            | Payload                                                   | Result                        |
| ---------------- | --------------------------------------------------------- | ----------------------------- |
| P0 control       | 2 backticks, Markdown, benign URL                         | 200 accepted                  |
| **P1 isolation** | 0 backticks in text, **no `parse_mode`**, backtick in URL | **200 accepted**              |
| **P2 isolation** | 1 backtick in text, Markdown, benign URL                  | **400** cannot parse entities |
| P3, Variant A    | 3 backticks, Markdown, backtick URL                       | **400** cannot parse entities |
| P4, Variant B    | 4 backticks, Markdown, backtick URL                       | **200 delivered**             |

The button URL carries the backtick in both real variants, so P3's failure on its own would be readable two ways: the entities failed, or Telegram rejected the URL. P1 and P2 settle it. P1 shows a backtick in a button URL is accepted when no parse mode is set, so the URL is not a failure cause. P2 shows one unbalanced entity is sufficient to fail on its own, with a benign URL present. P3's failure is therefore attributable to entity parsing rather than to URL rejection.

**Variant A, notification suppression.** An odd number of backticks leaves the entities unbalanced, Telegram returns 400, and the delivery is marked failed by [`services/telegram/utils.ts:82-89`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/telegram/utils.ts#L82-L89) and [`services/delivery/telegram.ts:94-107`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/delivery/telegram.ts#L94-L107). The victim silently receives no expiry warning for that name. This variant needs one character in the label.

**Variant B, content injection.** An even number balances the entities and the message delivers. Telegram's own parse of the delivered message, from the API response:

```
code     offset  40  len  5   -> "renew"
mention  offset  79  len 12   -> "@ensrenewals"
code     offset 110  len  5   -> "x.eth"
bold     offset 131  len  7   -> "30 days"
```

The decisive detail is the `mention` entity. **Telegram does not create mention entities inside a code entity.** So the presence of a mention entity at offset 79 is itself proof that the attacker's text sits outside the code span. That argument holds without reference to any screenshot.

### 5. The rendering hides the mechanism from the victim

The backticks are consumed as delimiters and are **invisible** in the delivered message. The victim sees:

```
renew-SECURITY-NOTICE-from-ENS-message-@ensrenewals-to-keep-this-name-x.eth
```

There is no visible metacharacter and no cue that the name is malformed. The template's own bold formatting survives intact, so the message does not look broken. The Renew Now button rendered on both P1 and P4, so the backtick-carrying URL produced a working inline keyboard rather than being stripped.

This removes the obvious mitigation. A user cannot notice something odd about the name, because nothing odd is displayed.

## Impact Details

An attacker authors arbitrary text and formatting inside a notification that the victim's Telegram client attributes to the official ENS bot, and places a tappable mention in it pointing at an account of their choosing. Separately, and with a single character, the same defect silently suppresses expiry warnings for a name the victim owns.

### Why this is not a duplicate of a known issue

**EXP-INPUT-005**, *ENS name validators accept homoglyphs, bidi and control characters*, is the nearest disclosed entry. The disclosure scopes it to *homograph-style display confusion in our UI rather than incorrect resolution*. This finding differs on four axes.

* **Medium.** EXP-INPUT-005 concerns the app's own UI. This concerns a third-party messenger, rendered by Telegram's parser rather than by the app.
* **Mechanism.** EXP-INPUT-005 is visual confusability, characters that look like other characters to a human. This is parser metacharacters, characters that are control syntax to a Markdown parser. A homoglyph does not close a code span. A backtick does.
* **Consequence.** EXP-INPUT-005 produces mistaken identity of a name. This produces attacker-authored content attributed to ENS, and separately a suppressed warning.
* **Component.** EXP-INPUT-005 is scoped to the validators in the two apps. This is `workers/api-worker`, which those validators never touch, and which is fed by the indexer rather than by a user-facing input.

**EXP-INPUT-009**, the push service worker opening a backend-supplied URL, is a different channel and a different sink. I note that [`templates/push.ts:34`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/services/delivery/templates/push.ts#L34) builds the same URL, so a push-channel variant of this would dedup into EXP-INPUT-009. **I am claiming the Telegram channel only.**

**EXP-GAP-006** is telemetry egress to a third party. **R2-05** is webhook ingress. **QA-08** is raw error text in the UI. None touches outbound message rendering.

### Why this is not intended behaviour

The reading to rule out is that ENS names are trusted here on purpose, because they are expected to be normalized before they ever reach a template. Two artifacts in this codebase contradict that reading, and a third fact falsifies the premise it rests on.

**The codebase treats normalization as load-bearing everywhere else.** The apps' shared `Domain` fragment requests **both** `name` and `normalizedName`, and the render paths prefer the normalized field. The indexer exposes `isNormalized` as a separate boolean precisely so a consumer can branch on it. The worker's own query requests **only `name`**, so it cannot branch on normalization even in principle. That is not a decision to trust the raw value, it is the absence of one.

**Every other rendering surface in this codebase escapes by construction.** The apps render through React, which escapes text nodes automatically, so the question of a metacharacter reaching a parser never arises there. The Telegram path is the one place where a string is handed to a parser as syntax rather than as text, and it is the one place with no escaping. The defect exists precisely where the framework stopped doing the work for them.

One more point, because it forecloses the strongest version of the intent argument. Even if names were assumed normalized upstream, that assumption is falsifiable: the indexer returns `normalizedName: null` and `isNormalized: false` for these names and hands back the raw label in `name`, and the production endpoint returns such rows in the worker's own query shape with no filter set. So the assumption, if it was made, is not true of the data the worker actually receives.

## References

* Template: `workers/api-worker/src/services/delivery/templates/telegram.ts`
* Send path: `workers/api-worker/src/services/delivery/telegram.ts`, `workers/api-worker/src/services/telegram/utils.ts`
* Indexer query: `workers/api-worker/src/services/expiry-discovery/indexer.ts`
* Endpoint pin: `workers/api-worker/wrangler.jsonc:99`
* Recipient resolution: `workers/api-worker/src/queues/event-ingestion.ts`
* Label registry emitting the raw label: `0x532cd0cc4ac0793d838f71a67d29b2d790d18777` on Sepolia
* Indexer endpoint: `https://graphql.ens.dev/`

## Link to Proof of Concept

<https://gist.github.com/chipsforbeer/17f45c34253497c7d5dd694f6fcabb96>

## Proof of Concept

Everything below was demonstrated with researcher-controlled artifacts on the program's designated testnet.

* Two `.eth` names were registered on Sepolia by the bounty test wallet `0xA3bF...F654`. They exist only to demonstrate that a backtick survives label registration. They hold no records.
* The label `renew` + backtick + `-SECURITY-NOTICE-from-ENS-message-@ensrenewals-to-keep-this-name-` + backtick + `x` is shaped like a lure **because that is what demonstrates the impact**. Nothing else about it is intended.
* `@ensrenewals` points at no account I own, control, or will register. I will not register it.
* The Telegram test ran in a private chat between a throwaway bot and its own creator. **No third party received any message at any point.** The bot was revoked immediately afterwards.
* No ENS asset was involved in the Telegram probe.
