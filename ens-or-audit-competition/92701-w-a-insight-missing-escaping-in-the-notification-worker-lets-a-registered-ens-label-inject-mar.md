# 92701 w a insight missing escaping in the notification worker lets a registered ens label inject markdown into official ens notifications placing an attacker chosen link inside messages delivere&#x20;

**Submitted on Sep 11th 2026 at 17:26:56 UTC by @Nyko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92701
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

The expiry-notification pipeline in `workers/api-worker` takes the `name` field returned by the ENS indexer and places it, unmodified, into the Telegram message it sends with `parse_mode: 'Markdown'`, into that message's inline button URL, and into the web-push body, tag and click URL. The worker never normalizes or escapes the label at any point. The indexer returns unnormalized labels verbatim, and the .eth registrar controller accepts any label of three or more characters, so a label may contain a backtick, brackets and parentheses. A label such as ``a`[Renew now](https://evil.example)`b`` closes the template's code span and leaves a **live Markdown link** in the message body. The message is delivered by the official ENS notification bot to the name's owner and to every user who has favourited that name.

The injected content is stored on chain in the label itself, so it is re-rendered on every notification about that name, to every recipient, indefinitely. That is persistent injection, not reflected.

**On which commit these line numbers refer to,** since the program page names a different hash. Every citation below was asserted programmatically against `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, the HEAD of the in-scope `audit-comp-ready` branch of `immunefi-team/audit-comp-ens`, with the clone verified unmodified (`git status` clean). That branch reaches back to `cda79ac`, *"fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86"* — the audited revision the program page states. The only changes after the fork point are four dependency manifests (`apps/manager/package.json`, `packages/dqa-server/package.json`, `workers/api-worker/package.json`, `pnpm-lock.yaml`, from a single `hono` bump). **None of them is a file cited in this report**, so every source file referenced here is identical to the audited revision.

## Vulnerability Details

**1. The name is taken from the indexer as-is.** `src/services/expiry-discovery/indexer.ts` selects `name` (`:21`) and copies it into the event (`:138`). The response schema (`:46-56`) validates `name` as a bare string. There is no normalization anywhere in the worker. The indexer schema exposes `normalizedName` and `isNormalized`; the worker reads neither.

**2. It reaches the message unescaped.** `src/queues/event-ingestion.ts` stores `name: event.name` in the notification payload at `:202` and `:227`. That value is the raw indexer string — the only `toLowerCase` calls on this path are applied to owner addresses (`:83`, `:128`, `:178`), never to the name.

`src/services/delivery/templates/telegram.ts:34-35` builds the message text, wrapping the name in a pair of single backticks (a Markdown code span) followed by the day count in asterisks. `:44` builds the inline button URL by appending the same name to the app address. `:39` sets `parseMode: 'Markdown'`, which becomes the Bot API's `parse_mode` at `src/services/delivery/telegram.ts:71` (`parse_mode: message.parseMode`).

`src/services/delivery/telegram.ts:68-72` passes `text`, `parse_mode` and `reply_markup` straight to `sendMessage`; `src/services/telegram/utils.ts:67` serializes the body with `JSON.stringify` and `:209-230` passes the button URL through unchanged. Nothing escapes.

`src/services/delivery/templates/push.ts:31-36` and `:43-47` place the raw name in the push body, the notification `tag`, and the click URL (`data.url`).

**3. Legacy Markdown makes the backtick load-bearing.** The Bot API documents legacy `Markdown` (core.telegram.org/bots/api, "Markdown style"): the syntax includes `[inline URL](http://www.example.com/)` and `` `inline fixed-width code` ``; "To escape characters '\_', '\*', '\`', '\[' outside of an entity, prepend the character '' before them"; and "Escaping inside entities is not allowed, so entity must be closed first and reopened again". The template puts the name **inside** a code entity and escapes nothing. A name containing a backtick closes that entity early, and everything after it is parsed as Markdown until the next backtick. Balanced backticks keep the message valid, so Telegram accepts and delivers it rather than rejecting it — which is what makes this a delivered link rather than a failed send.

**4. Unnormalized labels are a real state, and this repository says so four times.** The indexer the worker is configured against (`ENS_INDEXER_GRAPHQL_URL`, `wrangler.jsonc:99`) exposes both `normalizedName` (`packages/indexer/graphql.gen.ts:103`) and `isNormalized` (`:97`), and the Domain fragment selects `normalizedName` (`:1103`). The worker reads neither. ENS's own code then states the hazard directly:

* `apps/manager/src/features/dashboard/components/NamesTable.tsx:84-86` — *"Use the canonical normalized name for the on-chain label — the display name may be unnormalized (e.g. mixed case) and would hash to the wrong label"*, and it splits display from on-chain use accordingly. That is the split the notification templates do not make.
* `apps/portal/src/routes/$name/token.tsx:352` ships a user-facing **"Not Normalized"** badge, so the Explorer renders this state as an ordinary occurrence.

**Disclosed against myself:** `e2e/helpers/mock-indexer.ts:71` sets `normalizedName: d.name`, so ENS's own end-to-end mock never produces the divergence. That is test convenience, not evidence the state cannot occur — the two citations above are the project asserting that it does — but I would rather state it than have it raised.

**5. Such a label is registrable.** In `ensdomains/ens-contracts` (read at pin `55b0eb7`, cited as context — the contracts are out of scope for this competition and no contract defect is claimed):

```solidity
// contracts/ethregistrar/ETHRegistrarController.sol:191-193
function valid(string calldata label) public pure returns (bool) {
    return label.strlen() >= 3;
}
```

and `contracts/utils/StringUtils.sol:8-28` counts UTF-8 code points by inspecting each leading byte, applying no character filter of any kind. The only constraint on a .eth label is therefore a length of three or more code points. A backtick, a bracket and a parenthesis are all accepted.

**6. The same worker already rejects these exact characters — on the surface it did guard.** `src/app/routes/names/index.ts:13`:

```js
const INVALID_ASCII_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
```

applied at `:34` inside `isValidName`, which the public `/names` route enforces at `:47`. That class contains the backtick, both brackets, both parentheses and the asterisk — precisely the characters this injection needs, and nothing else in it matters here. The same worker, one directory away, embeds those characters unescaped into a message it signs with the official bot token. The guard is already written and already trusted; it is simply not on this path.

## Impact Details

**A link the attacker controls, inside a message the recipient has every reason to trust.** The proof of concept below renders, from the unmodified template files, the exact text the worker hands to `sendMessage`:

```
⚠️ *Domain Expiration Alert*

Your domain `a`[Renew now](https://evil.example)`b.eth` will expire in *7 days*.

Don't forget to renew your domain!
```

The backtick count preceding the link is even, so the link sits outside any code span and legacy Markdown renders it as a clickable "Renew now" pointing at a host the attacker chose. The message arrives from the official ENS notification bot, in the channel the user enrolled in precisely to be told about their names.

**The recipient takes no action to become a target.** A .eth name is an ERC-721 token, and a transfer needs no consent from the receiver. An attacker registers a name whose label carries the payload, transfers the token to the victim's address, and the victim is now the owner of a name whose expiry notifications carry the attacker's text and link. Delivery then follows the ordinary schedule. Recipients are not limited to the owner: `collectRecipientsForEvent` (`event-ingestion.ts:78-85`, `:198`) also resolves every user who has favourited the name, so one hostile label reaches every watcher of it.

**Why this is the persistent line and not the reflected one.** The payload is not derived from a request parameter. It is stored in the ENS label on chain, read back by the indexer, and re-rendered on every notification for that name, to every recipient, for as long as the name exists. Nothing the victim does clears it. The program's Medium line covers *reflected* injection — "Reflected HTML Injection, Loading external site data" — and the High line covers *persistent* injection, including "HTML injection without JavaScript" and "Replacing existing text with arbitrary text". Both describe this: the label replaces the notification's text with text the attacker chose, and the markup it injects renders as a link without any JavaScript.

**Addressing the one phrase in that line that does not obviously fit.** The High line says "on the target application", and this message renders in Telegram and in the OS notification surface rather than in a page of the Manager or the Explorer. The declared asset for this report is `workers`, which is an in-scope application in its own right, and it is the worker that composes the injected text and signs the delivery with the official bot token. The defect, and every line of the fix, is in `workers`. I am stating the tension rather than leaving it for review: if the phrase is read to require rendering inside the two front-end apps, the honest fallback is the Medium injection line — but that line says "reflected", and this is not reflected.

**Second consequence, reported as a consequence.** A label that unbalances the backticks makes Telegram reject the message, which suppresses the expiry alert for that name entirely. The user is not told their name is expiring. This is the same root cause and the same fix; it is noted for completeness and is not the impact being claimed.

**What I am not claiming.** The inline button URL stays on the `app.ens.domains` origin, so this is not an open redirect and I do not claim one. And although the same unescaped `payload.name` flows into the `name-transferred` templates for Telegram, push **and email** (`templates/telegram.ts:51`, `templates/push.ts:41`, `templates/email.ts:31-32`), nothing in the worker currently constructs a `name-transferred` notification — `event-ingestion.ts:225` only ever produces `kind: 'name-expiry'`. Those three templates share the defect and should share the fix, but they are not reachable today and I am not claiming them.

## Why this is not a published known issue

**The program's own out-of-scope text carves this class back in.** Under Websites and Apps: "Reflected plain text injection (e.g. url parameters, path, etc.) — *this does not exclude reflected HTML injection with or without JavaScript, or persistent plain text injection*". This is persistent, it is markup rather than plain text, and it produces a rendered link.

**Not EXP-INPUT-005.** That entry concerns the *name validators* in the Manager and Explorer accepting homoglyphs, bidi and control characters, and its stated impact is explicitly "homograph-style display confusion in our UI rather than incorrect resolution". This finding needs no deceptive characters — a backtick, a bracket and a parenthesis are plain ASCII — it is in the `workers` package where no validator sits on the path at all, and its outcome is a delivered link, not display confusion. The remediation implied by EXP-INPUT-005 (fixing the front-end validators) does not touch `templates/telegram.ts`.

**Not R2-02.** R2-02 is the absence of a Content-Security-Policy on the Manager, and it states that "an actual injection sink is a separate finding". This is an injection sink, and no CSP governs a Telegram message in any case.

**Not EXP-INPUT-009.** That entry is the push service worker opening a backend-supplied URL without validating its scheme or origin, and it notes it "requires control of, or injection into, the notification payload". This report is that injection, but it is not that finding: the click URL the label reaches is built as `https://app.ens.domains/${name}`, so the label lands in the *path* and cannot change the origin. I verified that and am not claiming an escalation of EXP-INPUT-009.

**Not R2-04, not R2-05, not QA-08.** R2-04 is the Explorer's OG-image worker fetching an avatar URL server-side; R2-05 is the SendGrid webhook's signature check failing open; QA-08 is raw error text overflowing its container in the apps. None concerns the notification templates, the indexer `name` field, the Telegram parse mode, or the push payload.

## Recommended fix

Read `normalizedName` (or check `isNormalized`) from the indexer and drop or hash-encode names that are not normalized. Escape the label for the chosen parse mode, or move to `MarkdownV2`/`HTML` with proper escaping. URL-encode the path segment of the button and push click URLs. Apply the same rule to every delivery template, including the three `name-transferred` templates that are not reachable today. The worker's own `isValidName` in `app/routes/names/index.ts` is the rule it already trusts elsewhere.

## References

* `workers/api-worker/src/services/expiry-discovery/indexer.ts` — `:21`, `:46-56`, `:138`
* `workers/api-worker/src/queues/event-ingestion.ts` — `:78-85`, `:198`, `:202`, `:225`, `:227`
* `workers/api-worker/src/services/delivery/templates/telegram.ts` — `:34-35`, `:39`, `:44`, `:51`
* `workers/api-worker/src/services/delivery/templates/push.ts` — `:31-36`, `:41`, `:43-47`
* `workers/api-worker/src/services/delivery/templates/email.ts` — `:31-32`
* `workers/api-worker/src/services/delivery/telegram.ts` — `:68-72`
* `workers/api-worker/src/services/telegram/utils.ts` — `:67`, `:209-230`
* `workers/api-worker/src/app/routes/names/index.ts` — `:13` (`INVALID_ASCII_CHARS`), `:34`, `:47`
* `packages/indexer/graphql.gen.ts` — `:97`, `:103`, `:1103`
* `apps/manager/src/features/dashboard/components/NamesTable.tsx` — `:84-86`
* `apps/portal/src/routes/$name/token.tsx` — `:352`
* `e2e/helpers/mock-indexer.ts` — `:71` (disclosed against this report)
* `ensdomains/ens-contracts` @ `55b0eb7` — `contracts/ethregistrar/ETHRegistrarController.sol:191-193`, `contracts/utils/StringUtils.sol:8-28` (context for registrability; no contract defect claimed)
* `workers/api-worker/wrangler.jsonc` — `:99`
* Telegram Bot API, "Markdown style": <https://core.telegram.org/bots/api#markdown-style>
* Frozen revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

## Proof of Concept

## Proof of Concept

**What this establishes:** that the worker hands Telegram a message whose text contains an attacker-chosen link, rendered outside any code span, from a label that is registrable on chain. The output below is produced by executing the two **unmodified** template files from the frozen revision — nothing is reimplemented.

No message was sent to Telegram, no push was delivered, and no live system was touched while preparing this report. The proof renders the request body the worker constructs.

### Run it

1. From a checkout of the frozen revision, copy `workers/api-worker/src/services/delivery/templates/telegram.ts` and `templates/push.ts` next to `poc.ts` (attached).
2. `node --experimental-strip-types --no-warnings poc.ts` — Node 22, no installs.

### Case A — a label the indexer returns verbatim today

Renders the payload for `test/name.eth`. Output: `parse_mode` is `Markdown`, and the button URL is `https://app.ens.domains/test/name.eth`. This establishes that unnormalized labels reach the template unchanged, using a name that exists on chain now.

### Case B — the injected link

Renders the payload for the label ``a`[Renew now](https://evil.example)`b``. The message text handed to `sendMessage`, with the backticks shown as they appear in the string:

```
⚠️ *Domain Expiration Alert*

Your domain `a`[Renew now](https://evil.example)`b.eth` will expire in *7 days*.

Don't forget to renew your domain!
```

The inline button URL for the same case is `https://app.ens.domains/a`[Renew now](https://evil.example/)`b.eth` — the raw label, including the link text, lands in the URL path. It stays on the `app.ens.domains` origin, which is why no redirect is claimed.

The script asserts, and all seven assertions pass:

1. the link appears verbatim in the text;
2. the number of backticks preceding the link is **even**, so the link is outside any code span and legacy Markdown renders it as a link;
3. the backticks are **balanced**, so Telegram accepts the message rather than rejecting it;
4. `parse_mode` is `Markdown`;
5. the push body carries the raw label;
6. the push `tag` carries the raw label;
7. the push click URL carries the raw label.

### Delivering it to a chosen recipient

1. Register a .eth name whose label carries the payload. `ETHRegistrarController.valid()` requires only three or more characters, and `StringUtils.strlen` counts UTF-8 characters, so a backtick, bracket and parenthesis are all accepted.
2. Transfer the ERC-721 to the victim's address. No consent from the receiver is required.
3. The victim now owns a name whose expiry notifications carry the payload. `collectRecipientsForEvent` (`event-ingestion.ts:78-85`) resolves the owner, so the victim is a recipient; it also resolves every user who favourited the name, so any watcher receives it too.
4. At the next expiry stage for that name, the official ENS bot delivers the message rendered above, with the attacker's "Renew now" link.

### Note on method

Established by reading and executing the shipped source at the frozen commit. The indexer facts in §4 come from read-only GraphQL queries against the public indexer the worker is configured to use — no write, no authenticated call, and nothing sent to Telegram, to SendGrid, or to any user.
