# 92445 w a insight telegram notification markdown injection via attacker controlled ens name

**Submitted on Sep 9th 2026 at 21:04:18 UTC by @Antivenom for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92445
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Injection of malicious HTML or XSS through metadata

## Description

The api-worker renders ENS name strings into Telegram notification messages using Telegram's Markdown mode (`parse_mode: 'Markdown'`) with no escaping, truncation, or character validation of the interpolated name. ENS names are attacker-controlled on-chain metadata: ENS v2 permits registering subnames with arbitrary 1-255 byte string labels (no ENSIP-15 normalization is applied to subname labels, and v2 labels are stored as plain strings rather than label hashes), so an attacker can create a name whose label contains Telegram Markdown syntax (backticks, square brackets, parentheses). When such a name flows into the name-expiry notification, the raw string is interpreted as Markdown by the Telegram API, letting the attacker inject a clickable link, mention, or arbitrary formatting into the official ENS bot message delivered to the victim's Telegram.

The injected Markdown is constructed to be perfectly balanced (no unclosed spans), so the Telegram API accepts the message and renders the attacker's link (verified below — the message cannot be rejected for malformed Markdown).

## Affected Asset

workers/api-worker (in scope)

## Location (all line numbers verified against the audit-comp branch)

* Sink: `workers/api-worker/src/services/delivery/templates/telegram.ts:34-35` — name-expiry template:

  ```
  const ownerText = payload.isOwner
    ? `Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\nDon't forget to renew your domain!`
    : `The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.`
  ```

  `payload.name` is interpolated raw — no escaping.
* Same sink, transfer template: `workers/api-worker/src/services/delivery/templates/telegram.ts:55-56` (`Your domain \`${payload.name}\` has been transferred.`) with` parseMode: 'Markdown'\` at line 58.
* `parseMode: 'Markdown'` set at `telegram.ts:39` (expiry) — Markdown parsing enabled for the message.
* Delivery: `workers/api-worker/src/services/delivery/telegram.ts:61-73` — message built from `deliveryJob.notification.payload as AnyPersonalNotificationPayload` (a bare cast, no runtime validation) and sent via `sendMessage` with `parse_mode: message.parseMode` (line 71). No escaping, sanitization, or truncation anywhere in the delivery path (verified by grep across `workers/api-worker/src`).
* Schema (only validation gate on the name): `packages/shared-schema/src/notifications/kinds/name-expiry.ts:8` — `name: v.string()` — type check only, no charset, no length cap, no pattern. Same for `name-transferred.ts:9`.
* Event ingestion: `workers/api-worker/src/types/events/index.ts:8` — `name: v.string()`; the queue parses incoming events with this schema only (`queues/event-ingestion.ts:525`). Names pass through byte-for-byte; the only normalization in the pipeline is `event.owner.toLowerCase()` (addresses, `event-ingestion.ts:82-83`) — names are never touched.
* Name source: `workers/api-worker/src/services/expiry-discovery/indexer.ts:13-27` — queries the official ENS indexer `https://graphql.ens.dev/`:

  ```
  domains(where: { expiry_gt: $cursor, expiry_lte: $upper_bound }) { name expiryDate owner { id } }
  ```

  and maps `domain.name` straight into the event (`indexerResponseSchema` = `name: v.string()`, no transformation).

## Proof of Concept

## Steps to Reproduce

### 1. On-chain (the vector — ENS v2 subnames with arbitrary labels)

ENS v2 (live) registries expose `register(string label, address owner, IRegistry subregistry, address resolver, uint256 roleBitmap, uint64 expiry)` (Permissioned Registry, `docs.ens.domains/ensv2/`). The only label constraint is size — "Labels are validated for size before registration" (1-255 bytes). No ENSIP-15 normalization is applied to labels by the registry. v2 subnames are stored as plain strings and emitted verbatim in `LabelRegistered(tokenId, labelHash, label, owner, expiry, sender)`.

Attack setup:

1. Register any name in ENS v2 (e.g. `attacker.eth` via the ETH Registrar) — or use any name you control.
2. Point a subregistry at it (`setSubregistry`) to enable subname registration.
3. Register a subname whose label is the Markdown payload (41 bytes, within the 1-255 limit):

   ```
   label = "x`[Renew now](https://evil.example.com)`x"
   register(label, victimAddress, ..., expiry = block.timestamp + 1 day)
   ```

   `owner` = the victim's address, so the victim becomes the owner of the maliciously-named subname. Full name as returned by the indexer: ``x`[Renew now](https://evil.example.com)`x.attacker.eth``

### 2. Indexer

The worker's expiry-discovery query (`expiry_gt: $cursor, expiry_lte: $upper_bound`) runs continuously against `graphql.ens.dev`, which indexes ENS v1 AND v2 domains including subnames, each with an `expiryDate` (v2 subname registrations carry expiry). The maliciously-named subname falls inside the polling window and is returned with its raw `name` string.

### 3. Notification creation

`queues/event-ingestion.ts` matches the returned `owner.id` to a user account (the victim's, since the attacker registered them as owner) and inserts a `name-expiry` notification with `payload.name = event.name` — the raw malicious string. (`v.string()` is the only validation.) No normalization anywhere.

### 4. Delivery — verified output (deterministic proof)

The template at `telegram.ts:34` renders the following exact message, which is sent to Telegram with `parse_mode: 'Markdown'`:

```
⚠️ *Domain Expiration Alert*

Your domain `x`[Renew now](https://evil.example.com)`x.attacker.eth` will expire in *7 days*.

Don't forget to renew your domain!
```

Tokenization of this message per Telegram Markdown rules (deterministic, reproduced locally):

```
literal : '⚠️ '
BOLD    : '*Domain Expiration Alert*'          (renders bold)
literal : '\n\nYour domain '
CODE    : '`x`'                                (renders inline code)
LINK    : '[Renew now](https://evil.example.com)'   <-- INJECTED, renders as a CLICKABLE LINK
CODE    : '`x.attacker.eth`'                   (renders inline code)
literal : ' will expire in '
BOLD    : '*7 days*'                           (renders bold)
literal : '.\n\nDon't forget to renew your domain!'
```

Every span is closed (zero unclosed backticks or asterisks), so the Telegram API accepts the message (no "can't parse entities" error) and renders the attacker-controlled link "Renew now" → `https://evil.example.com` inside the official ENS bot message. The victim sees the legitimate alert header, the injected phishing link, and the legitimate "Renew Now" button — a perfect spoof.

### 5. ON-CHAIN PROOF — executed on Sepolia (ENSv2 beta, Sep-09-2026)

The malicious subname was registered on-chain on the ENSv2 beta (Sepolia) deployment, proving the ENS v2 registry accepts arbitrary label strings and stores them raw:

* **Transaction:** `0xa74f3cdfd68d0fd95c92a52fb8a6761ff171bafeffed1c8782b0a33fbfc089eb` (Status: Success, block 11670088) — <https://sepolia.etherscan.io/tx/0xa74f3cdfd68d0fd95c92a52fb8a6761ff171bafeffed1c8782b0a33fbfc089eb>
* **Function:** `register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expires)` — MethodID `0x85f3e643`
* **Contract:** `0x4B66DE3438FcB2c02E108d6b94B7218CD18521fA` (UserRegistry acting as subregistry of the parent name `mdtest3811.eth`, itself registered on the Sepolia v2 `.eth` registry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`)
* **Registered label (verbatim, as stored on-chain):** ``x`[Renew now](https://evil.example.com)`x`` — 41 bytes, within the registry's 1-255 byte label limit
* **`LabelRegistered` event** (contract `0x4b66de...`): `label (string): x\`[Renew now](https://evil.example.com/)\`x`,` owner: 0x45262418909CE2b4c6394E6a56146dA4E5ebABB5`,` expiry: 1791572000\`
* **`Label` event** (labels registry `0x532cd0cc...`, the contract the indexer reads to construct human-readable names): same raw string — `label (string): x\`[Renew now](https://evil.example.com/)\`x\`
* Resulting full name as returned by the indexer: ``x`[Renew now](https://evil.example.com)`x.mdtest3811.eth``
* **Control contrast:** the same label was **rejected by the app UI** (`app.ens.dev`: "subname must be a single label - dots are not allowed") while the **contract accepted it without any normalization** — confirming the only gate is UI-side input validation, not the protocol or the indexer.

This proves: (a) arbitrary Markdown-bearing labels are registerable in ENS v2 with an expiry timestamp, (b) the label string the indexer uses to build `domain.name` is the raw attacker string, and (c) the worker's expiry-discovery query therefore returns this raw string into the notification pipeline described above.

### 6. Impact realization

The victim receives this message because they own (or watch) the maliciously-named name — the attacker registered them as owner with an expiring subname. The injected link displays the attacker's chosen text ("Renew Now") inside the trusted ENS bot message. ENS users are trained to click "renew" links from this bot; a fake renewal page can harvest wallet signatures/credentials.

## Impact

* Attacker-controlled clickable links, usernames/mentions and formatting injected into ENS-branded Telegram notifications, delivered by the official ENS notification bot to victims who own (or watch) attacker-registered subnames.
* The message passes no secondary validation — it originates from the bot itself, so the injected link appears as trusted ENS communication: a direct phishing/credential-harvesting vector against ENS users (e.g., fake renewal UI requesting wallet connection or signature).
* Preconditions are product-normal: the victim is an ENS notifications user (the product's purpose) with Telegram channel enabled and name-expiry notifications on — the attacker controls the rest end-to-end.
* Distinct from known issue EXP-INPUT-005: EXP-INPUT-005 concerns the Manager/Explorer app input validators accepting homoglyphs/bidi/control characters (a display-confusion issue in the apps' inputs). This finding is a missing output encoding in the worker's Telegram delivery pipeline — a different component, different data source (indexer → worker, not app forms), and a different root cause (no escaping of Markdown control characters at the sink).

## Root Cause

Telegram message text is built by string interpolation of attacker-controlled ENS name metadata (indexer-provided, never normalized) and sent with `parse_mode: 'Markdown'`; the only validation on the name field is `v.string()`; no Markdown escaping exists anywhere in the worker.
