# 91702 w a insight telegram expiry notifications render an attacker chosen link inside the official ens bot s message

**Submitted on Sep 3rd 2026 at 18:41:57 UTC by @debuhkzr for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91702
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

While looking at the notification delivery code in workers/api-worker, I found that the ENS label coming from the indexer gets dropped straight into a Telegram message with parse\_mode set to Markdown, with no escaping at all. Same thing happens with the inline button URL — no encodeURIComponent, nothing.

The name itself originates on-chain (whatever label someone registered), flows through the indexer as a plain v.string(), gets persisted as-is, and ends up interpolated directly into the message template. Since Telegram is told to parse this as Markdown, any Markdown metacharacters in the label get interpreted rather than shown as plain text.

I spent some time making sure this wasn't just a theoretical parser issue before writing it up.

First, I checked which parse mode is actually in use, because Telegram has two — legacy Markdown and MarkdownV2 — and they behave pretty differently. The type in templates/telegram.ts only allows 'Markdown' | 'HTML', so MarkdownV2 isn't even reachable here; grepping the whole codebase for it turns up nothing outside my own test file. That said, I don't think the specific leniency of legacy Markdown is really the load-bearing part of this. The label an attacker registers is entirely their choice, so they'd just pick a payload where every backtick and bracket is balanced — which parses fine under either mode. I checked this specifically: the payload I used has 4 backticks (two closed code spans), 4 asterisks (two closed bold spans), and one well-formed link. Nothing malformed about it.

Second, I wanted to be honest about who can actually receive this, since transferring a name to someone doesn't automatically mean they get pinged. Going through the code, a victim has to already: have signed into the app with that wallet (creates the users row), have linked and verified a Telegram channel, and have explicitly turned on the "name expiring" notification setting, which defaults to off. So this isn't "any name-transfer target" — it's specifically users who've opted into these notifications. What is true zero-interaction is the attacker's side: transferring an ENS name to someone requires no consent from them at all, so getting a target set up doesn't need them to do anything.

Third, and this is the part I think matters most for taking this seriously — I didn't just trust that Telegram's API would actually accept the payload, I tested it against the real api.telegram.org with a disposable bot (revoked right after testing). Sent two messages:

The first was just the text body with parse\_mode Markdown. Telegram came back with ok: true, and the response's entities array showed it had parsed my label into an actual text\_link entity pointing at my attacker URL — so the visible word "Renew" in the message is a genuine clickable link to an external site, not some inert text.

The second test included the same label in a reply\_markup button URL. Telegram accepted that too, ok: true, no BUTTON\_URL\_INVALID error, even with backticks and brackets sitting in there. Worth being precise about this one though — the button's URL string still starts with <https://app.ens.domains/>..., so the scheme and authority don't change. The attacker's characters just end up mangling the path. So I'm not claiming this is a second link to an external site, only that the unescaped label flows into the button construction too, which is a related but smaller issue than the main text\_link finding.

So the real, demonstrated impact here is: a link appears inside a message sent by ENS's actual verified Telegram bot, and it goes to wherever the attacker wants. I'm not claiming credential theft or fund loss directly happened in my testing — that would need the victim to click through and interact with whatever page the attacker put up, which is outside what I tested. I just want to be upfront that what I proved is the injection and delivery, not what happens after someone clicks.

On duplicates: there's a disclosed item, EXP-INPUT-009, about the push notification service worker opening an unvalidated URL from notification data. I don't think this overlaps — that one is client-side, in the browser's push-sw\.js, triggered by a click on a push notification. What I'm describing is server-side, in the worker's own template rendering before anything reaches a client. Different code path, different fix. I also checked whether my finding opens up EXP-INPUT-009 as a new attack surface and it doesn't — the push flow builds its URL with a fixed <https://app.ens.domains/> prefix regardless of what's in the label, so there's no origin escape there either way.

Also checked it against the CSP-related disclosure (R2-02), which explicitly says an actual injection sink counts as a separate finding from just missing CSP headers, so that doesn't cover this either.

Where this actually lives:

* Where the name comes from: workers/api-worker/src/services/expiry-discovery/indexer.ts line 43
* Payload type: packages/shared-schema/src/notifications/kinds/name-expiry.ts line 8
* Persisted: workers/api-worker/src/queues/event-ingestion.ts lines 226-231
* The actual sink for the text: workers/api-worker/src/services/delivery/templates/telegram.ts lines 34-35
* The button URL sink: same file, line 44
* Where parse\_mode gets set and forwarded: telegram.ts line 39, delivery/telegram.ts line 71

For fixing it — the safest option is probably to just not use Markdown formatting for anything that includes an untrusted label, and fall back to plain text. If formatting is genuinely needed, escape properly for whichever parse mode is chosen. Same treatment should go on the button URL with encodeURIComponent, and honestly the same pattern probably needs checking in the transfer-notification and push templates too, since they likely have the same shape of bug.

## Link to Proof of Concept

<https://gist.github.com/Devendrakumarsinha/07b2dd342902056d2d2b2fecef0fde74>

## Proof of Concept

I wrote a local test file for the template logic, and separately ran two live requests against the actual Telegram Bot API to confirm real-world behaviour, not just my own assumptions about how the parser works.

Local test: File: workers/api-worker/src/services/delivery/templates/RESEARCH\_telegram-injection.test.ts (new file only, nothing in production code touched)

cd workers/api-worker pnpm exec vitest run src/services/delivery/templates/RESEARCH\_telegram-injection.test.ts --reporter=verbose

4 tests, all passing: confirms legacy Markdown is what's actually configured, confirms the injected payload is entity-balanced so it won't just break parsing, confirms the button URL inherits the raw label, and a control case with a normal label to show nothing weird happens when there's nothing malicious in the input.

Live API test (the part I think actually matters most): I created a disposable Telegram bot through BotFather purely for this test, sent two requests to api.telegram.org, and revoked the bot token immediately afterward.

Payload label used: a`[Renew](https://ens-renew.example/drain)`b (48 bytes, well under the registrar's 255-byte limit)

Request 1 - plain sendMessage with parse\_mode Markdown: Response came back ok: true, and the entities array contained: {"offset":41,"length":5,"type":"text\_link","url":"<https://ens-renew.example/drain"}>

That's Telegram itself confirming the word "Renew" in the delivered message is a live link to my attacker URL.

Request 2 - same message plus a reply\_markup inline button, with the button's URL also containing the raw unescaped label: Also ok: true, no rejection, button URL accepted as submitted: "reply\_markup":{"inline\_keyboard":\[\[{"text":"Renew Now","url":"<https://app.ens.domains/a`[Renew>]\(<https://ens-renew.example/drain)`b.eth"}]]}>

As noted above, I want to be precise that this second one doesn't prove a second external navigation path — the authority stays app.ens.domains, only the path gets mangled by the unescaped characters. What it does prove is that the raw label reaches this URL construction completely unescaped, which is worth fixing even though I'm not claiming it as an independent phishing vector on its own.

I also checked, out of caution, whether a label like this could actually exist on-chain in the first place — if the registrar rejected such characters this whole report would be moot. I'm not filing anything against the contracts themselves since they're out of scope for this competition, I only checked this as feasibility evidence. Cloned ensdomains/contracts-v2 locally at the commit the app is actually pinned to (8d1c893). The only validity check in StandardRentPriceOracle.getBasePrice() (lines 365-374) is on byte length, nothing about character content, and I couldn't find any additional check in either register() or renew() when I grepped for it. There's actually a comment directly on the contract saying "Does not check if normalized," so this isn't really ambiguous.

Screenshots and a screen recording of both the local test run and the live Telegram API responses are attached. The disposable bot token has been revoked and is redacted from what's shown.
