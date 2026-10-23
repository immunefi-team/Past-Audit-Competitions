# 92439 w a insight unescaped ens name in telegram expiry notification template enables persistent markdown link injection reaching third party users

**Submitted on Sep 9th 2026 at 20:04:53 UTC by @asian13 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92439
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.
  * Redirecting users to malicious websites (open redirect)

## Description

## Brief/Intro

Expiry-notification messages sent via the ENS Telegram bot (`parse_mode: 'Markdown'`) interpolate the event's `name` field directly into the message text with no escaping of Markdown metacharacters. An attacker can register an ENS name whose label contains a backtick, breaking out of the template's inline-code span and injecting a fully-formed, entity-balanced Markdown link with attacker-chosen display text and URL. The result is delivered as a normal, successfully-parsed message from the real, already-trusted ENS Telegram bot to every user who favourited the malicious name — not just the attacker.

## Vulnerability Details

`workers/api-worker/src/services/delivery/templates/telegram.ts` builds the expiry-alert text as:

```
const ownerText = payload.isOwner
  ? `Your domain \`${payload.name}\` will expire in *${daysLeft} days*.\n\n...`
  : `The domain \`${payload.name}\` you're watching will expire in *${daysLeft} days*.`
 
```

`payload.name` comes from `ExpiryEvent.name`, typed as an unconstrained `v.string()` in`types/events/index.ts`, and passed through `event-ingestion.ts`'s `collectRecipientsForEvent`with no normalization or character filtering anywhere in the pipeline(`services/expiry-discovery/indexer.ts` performs no sanitization either).

Because the name is wrapped in a single pair of backticks, a name containing its own backtickcloses that code span early. A crafted label such as:

```
evil` 🎁 [Claim your free ETH refund here](https://evil-attacker-site.example/phish) 🎁 `y
```

keeps every Markdown entity in the final message balanced (Telegram's Bot API rejects a messageoutright if entities are unbalanced), while injecting a complete, independent `[text](url)` linkas ordinary message text in between.

\*\*On-chain registerability independently confirmed (not just cited from the known-issues doc):\*\*we ran the project's own `e2e/scripts/seed-name.ts` against the real V2 ETHRegistry/registrarcontracts (the project's standard Anvil-forked-Sepolia harness — the same contracts the Managerapp itself talks to) and successfully completed a full commit-reveal registration for the label``evil`test`` — the registrar accepted the raw backtick with no rejection. This removes anydoubt that a malicious label of this shape is registerable, not merely a display-layerassumption.

## Impact Details

This is submitted as at minimum Medium ("open redirect"), but we believe it sits closer to High:

1. **Delivery through an already-trusted, first-party channel.** The injected link arrives fromthe real, official ENS Telegram bot, in the same message the user already relies on forlegitimate expiry alerts, directly alongside a genuine "🔄 Renew Now" button.
2. **One-to-many amplification.** A single malicious name registration reaches every account that ever favourites that name, indefinitely, each time an expiry-related event fires — astructurally worse blast radius than a classic single-URL open redirect.
3. **Directly analogous to the program's own High bucket** — "Injecting/modifying the staticcontent on the target application without JavaScript (persistent)" — just delivered via thisin-scope backend's bot-message channel rather than a web page.
4. **Zero-interaction targeting** — a victim never interacts with the attacker; favouriting apublic ENS name is sufficient.

Realistic end-to-end scenario: ENS's user base is by construction 100% wallet-connected. Anattacker links to a pixel-perfect clone of app.ens.domains's "Renew"/"Connect Wallet" flow, worded like the legitimate surrounding copy. The only thing distinguishing the malicious buttonfrom the real one, inside the same trusted-bot message, is the underlying URL — not visible in Telegram's UI without long-pressing the link. This is a realistic, low-friction on-ramp tofund-loss outcomes, though completing that chain requires a separate, out-of-scope phishing page.

This report also completes the missing precondition for the known **EXP-INPUT-009** issue: theidentical unescaped `payload.name` flows into the push-notification template (`services/delivery/templates/push.ts`, `data.url = https://app.ens.domains/${payload.name}`),which that known issue's own writeup flagged as requiring "control of, or injection into, the notification payload" as an unconfirmed precondition — now demonstrated end-to-end via the samepipeline. We further confirmed the registrar equally accepts `/` and `?` in a label, so a crafted name can direct a tapped push notification at an arbitrary existing in-app route with arbitraryquery parameters, not merely an arbitrary-looking path segment (we checked every Manager routeconsuming URL search params for a dangerous auto-triggered action; none currently exists, but weflag this for completeness and future-proofing).

Distinct from the already-known **EXP-INPUT-005** (scoped by the team to UI *display* confusion in the Manager/Explorer search input only) — this is a different code path (notificationtemplates) with a materially more severe consequence (phishing-link injection reaching thirdparties).

## References

* workers/api-worker/src/services/delivery/templates/telegram.ts
* workers/api-worker/src/services/delivery/templates/push.ts
* workers/api-worker/src/types/events/index.ts
* workers/api-worker/src/services/expiry-discovery/indexer.ts
* workers/api-worker/src/queues/event-ingestion.ts

## Proof of Concept

## Proof of Concept

1. Standalone reproduction (no monorepo build needed), `poc_telegram_markdown_injection.mjs` — a verbatim copy of the real `nameExpiryTemplate` function. Run with plain Node:

   const maliciousName = 'evil`🎁 [Claim your free ETH refund here](https://evil-attacker-site.example/phish) 🎁`y' nameExpiryTemplate({ name: maliciousName, expiryDate: Date.now()+5\*86400\_000, isOwner: false }).text

   Produces: "⚠️ *Domain Expiration Alert*\n\nThe domain `evil` 🎁 [Claim your free ETH refund here](https://evil-attacker-site.example/phish) 🎁 `y` you're watching will expire in *5 days*." — every Markdown entity pairs correctly.
2. Live-confirmed against the real Telegram Bot API: sent the exact message via `POST https://api.telegram.org/bot<token>/sendMessage` with `parse_mode=Markdown`. Telegram accepted it (HTTP 200) and its own response confirms the parsed entity list includes `{"type":"text_link","offset":47,"length":31,"url":"https://evil-attacker-site.example/phish"}` — Telegram's own server-side parser independently confirms the render, not just our prediction. Screenshot attached (evidence\_telegram\_markdown\_injection\_live.png) shows the rendered message: the alert title, the injected clickable phishing link, and the legitimate "Renew Now" button in the same message.
3. On-chain registerability confirmed via the project's own e2e harness:

   LABEL='evil\`test' STATE=active pnpm --filter @ens-apps/e2e seed:name

   Output: "\[makeName] ✅ registered evil\`test-.eth" — full commit-reveal registration succeeded against the real forked V2 registrar contracts, confirming the backtick-containing label is genuinely registerable, not a display-layer-only assumption.
4. Attack path: attacker registers a name containing the crafted label → any user who favourites the name (an ordinary product action) is added as a notification recipient via `collectRecipientsForEvent` → when the name approaches expiry, every recipient receives the crafted message from the real ENS Telegram bot.
