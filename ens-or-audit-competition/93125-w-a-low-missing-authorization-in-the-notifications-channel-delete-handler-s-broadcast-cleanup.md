# 93125 w a low missing authorization in the notifications channel delete handler s broadcast cleanup lets any authenticated user delete another user s sendgrid marketing contact config gated&#x20;

**Submitted on Sep 14th 2026 at 10:35:33 UTC by @FestiveSRT for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93125
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

An authenticated user can delete another user's SendGrid marketing contact. When an email channel is deleted, the DELETE handler's broadcast-cleanup path searches SendGrid for a contact by the channel's email alone and deletes the whole contact, with no check that the deleter ever owned or verified that email. Combined with a cross-user collision block that rejects only *verified* collisions, an attacker can pre-register a victim's email as the attacker's own *pending* channel, wait for the victim to independently verify that email (which enrolls it in the marketing list), then delete the attacker's own pending row. That removes the victim's entire SendGrid marketing contact and suppresses all marketing broadcasts to that user. The victim's own verified channel row is untouched, so nothing alerts them. In production this is gated by a single config flag (SENDGRID\_BROADCAST\_LIST\_ID): where that flag is set, the path is live and this is an exploitable cross-user setting change; where it is empty (as it ships in the repo) the path is inert and this stands as a latent authorization defect to fix before the integration is enabled.

## Vulnerability Details

Precondition (the exploit only works against a not-yet-verified victim): the attacker must hold a pending channel row for the victim's email before the victim verifies it. The cross-user collision block filters to already-verified rows only, so once the victim is verified an attacker can no longer register that address; an already-verified victim is immune. This is therefore a one-shot removal against a victim the attacker pre-registers.

Mechanism, every step cited into the pinned tree:

1. POST /notifications/channels/email (workers/api-worker/src/app/routes/notifications/channels/email.ts:29) is authenticated; it uses the caller's user\_id (email.ts:35) and accepts+normalizes the submitted email (email.ts:36-37).
2. The cross-user collision check filters to status = 'verified' ONLY (email.ts:59-72, filter at :63; the reject branch at :67 fires only when an *already-verified* row belongs to another user). So the attacker's PENDING row for the victim's email is accepted and upserted to the attacker (email.ts:75).
3. Row uniqueness is the composite (user\_id, channel, target) (workers/api-worker/src/core/database/schema/notifications.ts:74), NOT global on email, so the attacker and the victim can each hold a separate pending row for the same address.
4. The victim later verifies the victim's own token (email.ts:179-201). If SENDGRID\_BROADCAST\_LIST\_ID is set, the verify path adds the victim's email to the marketing list (email.ts:233-236 → addContactToList, workers/api-worker/src/services/sendgrid/contacts.ts:53-71).
5. The attacker deletes the attacker's OWN pending channel: DELETE /notifications/channels/:id (workers/api-worker/src/app/routes/notifications/channels/$id.ts:62), passing the ownership check on the channel row (the lookup is scoped to id AND user\_id — $id.ts:66-75), then deleting the row (:77-79).
6. The defect: the broadcast-cleanup ($id.ts:81-96; guard at :82-86, search+delete at :90-96) fires whenever channel === 'email' && target && SENDGRID\_BROADCAST\_LIST\_ID (flag at :85), with NO verified-status check and NO contact-ownership check. It searches the provider by email alone (searchContact, contacts.ts:113-153) and then deletes the ENTIRE contact (deleteContact, contacts.ts:223-232) — not merely one list membership.
7. Result: the victim's marketing contact is removed while the victim's DB channel row remains verified, so nothing alerts the victim.

Config gate: the same SENDGRID\_BROADCAST\_LIST\_ID flag gates both the marketing add on verify (email.ts:233) and the cleanup delete ($id.ts:85), so the defect is exploitable only where that flag is non-empty. Every configuration visible in the repo ships the flag empty: wrangler.jsonc:107 sets SENDGRID\_BROADCAST\_LIST\_ID: "" in the only shipped config (real BASE\_URL, template IDs, VAPID key) with no override anywhere, so at the shipped deployment the path is inert.

## Impact Details

Unauthorized deletion of another user's SendGrid marketing contact: the entire contact is removed (deleteContact, contacts.ts:223-232 — not one list membership), suppressing all marketing broadcasts to that user. It is achieved with permissionless authenticated attacker actions and no victim interaction at the deletion moment.

Selected impact and why it matches. This maps to the listed Medium bullet: "Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications". Deleting the victim's marketing-broadcast subscription is a non-sensitive notification-setting change; there is no already-connected wallet interaction; and the victim's single independent verify-click is the "up to one click of user interaction." The impact is in scope (a listed website/application impact) and demonstrable from the mechanism above.

Bounds (kept honest):

* No funds loss, no name loss, no account takeover, no general email suppression. Transactional/notification email is sent directly via /v3/mail/send (workers/api-worker/src/services/email/utils.ts:27) and does not depend on the marketing contact; only marketing-broadcast delivery is affected. The victim's own verified channel DB row is untouched.
* Ceiling (the triager's call, NOT self-claimed): the Critical bullet — "Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes" — is arguable on the reading that the victim does nothing at the deletion moment. The exploit as a whole requires the victim's verify-click, so the grade claimed is Medium; the ceiling is noted only so the triager can apply it if they weigh the deletion moment alone.
* High does NOT apply: the High bullets require changing a *sensitive* detail ("Changing sensitive details of other users … such as: Email") or disclosing confidential information ("Improperly disclosing confidential user information, such as: Email address"). This changes a non-sensitive setting and discloses nothing.
* Config-gated: exploitable only where SENDGRID\_BROADCAST\_LIST\_ID is non-empty; the shipped value is "". At the shipped configuration the path is inert and this is a latent authorization defect; the final tier should be set against the real deployment config, which triage alone can see.
* Distinctness: this is not the known SendGrid issue about the event webhook failing open when the verification key is unset. That is a different mechanism; this is authenticated channel-management cleanup in the DELETE handler.

## References

* Cross-user collision block (verified-only filter): workers/api-worker/src/app/routes/notifications/channels/email.ts:59-72 (filter to status='verified' at :63; reject branch at :67); authenticated caller user\_id at :35; email accept/normalize at :36-37; pending-row upsert at :75.
* Verify path + marketing add: email.ts:179-201 (verification), email.ts:233-236 (gated add) → addContactToList at workers/api-worker/src/services/sendgrid/contacts.ts:53-71.
* Composite row uniqueness (not global on email): workers/api-worker/src/core/database/schema/notifications.ts:74 (unique on user\_id, channel, target).
* DELETE handler + the defect: workers/api-worker/src/app/routes/notifications/channels/$id.ts:62 (route), :66-75 (ownership-scoped lookup), :77-79 (row delete), :81-96 (broadcast-cleanup: guard :82-86, flag :85, search+delete :90-96).
* SendGrid service functions: searchContact contacts.ts:113-153; deleteContact (whole-contact delete) contacts.ts:223-232; the list-scoped removal helper that should be used instead is removeContactFromList at contacts.ts:179.
* Transactional email is independent of the marketing contact: workers/api-worker/src/services/email/utils.ts:27 (/v3/mail/send).
* Config gate ships empty: workers/api-worker/wrangler.jsonc:107 (SENDGRID\_BROADCAST\_LIST\_ID: "").
* Programme impact relied on: the listed Medium website/application impact "Changing non-sensitive details of other users … such as: Enabling/disabling notifications" (Impact selection above).
* Audited revision / provenance: the audited tree is byte-identical to public competition repo immunefi-team/audit-comp-ens at commit cda79ac (root tree 8e0d79dac1ab4b4fdb80d6afed810087ae9f00ba); the Resources tab names revision 63772fd8, which cda79ac embodies (its commit message forks ensdomains/apps-monorepo at 63772fd8). The in-scope source CODE is unchanged through head 1c9b47f — only hono dependency-version bumps in manifests. All source citations above are file:line into this pinned tree.

## Recommended remediation

Do not authorize contact deletion from a pending/unverified channel. Before the destructive cleanup, require status === 'verified' AND check that no other user holds a verified channel for the same target; prefer the list-scoped removal helper (contacts.ts:179) over the global contact delete (contacts.ts:223-232).

Guardrails so the fix is precise and cannot itself misfire: (i) The "no other user holds a verified channel" query MUST filter status = 'verified' exactly. Do NOT broaden to any status, or an attacker's freely-created pending row for the target becomes a griefing lever that blocks the true owner's cleanup. (ii) It MUST exclude the channel being deleted (run it after the row delete, or add ne(userChannels.id, channelId)); otherwise the deleter's own verified row self-matches and legitimate cleanup never fires. Optionally gate on verified\_at IS NOT NULL rather than status === 'verified', so an owner deleting a channel that was verified and later moved to bounced/unsubscribed still gets its lingering list membership cleaned; a pending row has verified\_at = null, so the attacker stays fully blocked.

## Link to Proof of Concept

<https://gist.github.com/AFestive/23f4ed6f51b0e5da50295ddc6aaeb6dc>

## Proof of Concept

## Proof of Concept

Executed offline, in-memory, running the REAL pinned handlers unmodified — email.ts and $id.ts, the real requireAuth (hono/jwt) and injectDb middleware, the real services/sendgrid/contacts.ts, and the real drizzle schema — transpiled and dispatched as real Requests through the real router. Storage is in-memory SQLite behind drizzle's real pg-proxy query builder (only the storage engine is swapped Postgres→SQLite via a small dialect shim). Network is intercepted: globalThis.fetch is a SendGrid mock that THROWS on any destination other than api.sendgrid.com. This is a web-application PoC, so the external SendGrid service is mocked (required for a web2 target, not a deficiency); the mock faithfully implements the four SendGrid endpoints the code calls. No live system was touched.

Steps (two conditions plus a negative control):

1. Run A — exploit (flag = "test-list"): the attacker creates a pending row for the victim's email (HTTP 200). The victim verifies the victim's own email (200). The attacker's attempt to DELETE the victim's channel directly is rejected (404, ownership check). The attacker then deletes the attacker's OWN pending row (200): the SendGrid mock receives POST /v3/marketing/contacts/search/emails for the victim's email, then DELETE /v3/marketing/contacts?ids=… for the resolved contact id. The victim's marketing contact is GONE while the victim's channel row is still verified. VERDICT: PASS (exploit fires).
2. Run B — control (flag = "" read verbatim from wrangler.jsonc): the identical step sequence produces ZERO marketing requests and the victim's pre-seeded marketing contact is preserved, confirming the config gate. VERDICT: PASS (path inert when flag empty).
3. Negative control (fail-before / pass-after): a patched copy of $id.ts that adds channel.status === 'verified' to the cleanup guard is loaded through the same loader and composed with the real, unmodified email.ts. The identical exploit sequence becomes inert (0 marketing requests, victim contact preserved), proving the harness discriminates the real defect rather than the plumbing. VERDICT: PASS (fix closes it).

Reproduction: from the evidence directory, run `node harness.cjs exploit`, `node harness.cjs control`, and `node negctl.cjs`. Requires Node >= 22.13 (tested on 22.23.2; the harness uses node:sqlite). The harness transpiles and runs the real audited source, so the loader references the audited checkout by absolute path; set that path constant to your checkout of the pinned repo.

Full runnable harness (the exploit/control runner harness.cjs, the negative-control runner negctl.cjs, the transpiling loader, the Postgres→SQLite dialect shim, and the SendGrid mock) and the recorded exploit/control verdicts are provided as a secret Gist.
