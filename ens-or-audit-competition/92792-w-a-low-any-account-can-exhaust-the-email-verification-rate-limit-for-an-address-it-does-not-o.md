# 92792 w a low any account can exhaust the email verification rate limit for an address it does not own blocking that address s verification for an hour

**Submitted on Sep 12th 2026 at 10:32:20 UTC by @crypt0n1337 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92792
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
* Locking up the victim from login
* Cookie bombing, etc.

## Description

## Brief/Intro

The API worker's email-verification rate limiter is keyed on the **address being verified**, not on the account making the request:

```
workers/api-worker/src/services/notifications/email-verification-rate-limit.ts:29-30
KV_KEY.NOTIFICATIONS.EMAIL_VERIFICATION(normalizeEmailForRateLimit(email))
  ->  notifications:email-verification:<email>
```

with `EMAIL_VERIFICATION_RATE_LIMIT_MAX_SENDS = 3` per window. The counter is consumed by the two routes that send verification mail (`app/routes/notifications/channels/email.ts:126` and `app/routes/notifications/channels/$id.ts:173`) and neither the key nor the consumption carries the requesting `user_id`. Any account — a free SIWE login — can therefore spend another person's budget by naming their address, and the person whose address it is is then refused.

## Executed evidence

The real worker routes were driven in process (real Hono app, real rate limiter, real KV binding substituted only at the binding boundary, real SendGrid mail builder replaced by a counting stub). Verbatim from the run:

```
[B1] key written = ["notifications:email-verification:victim.person@example.com" ×3, …someone.else…]
[B1] victim-facing string = Too many verification emails sent to this address. Try again in 60 minutes.

[B2] attacker request 1 -> 200, one verification mail addressed to victim.person@example.com
[B2] attacker request 2 -> 200, one verification mail addressed to victim.person@example.com
[B2] attacker request 3 -> 200, one verification mail addressed to victim.person@example.com
[B2] victim request    -> 429 {"error":"Too many verification emails sent to this address. Try again in 60 minutes."}
[B2] mails actually sent to the victim's address on the victim's own request: 0
```

The window is clock-driven KV state, so the attacker can renew it: three requests per hour keep the victim's address permanently refused. The victim has no way to distinguish this from a transient error, and the refusal is identical in shape to the refusal the address owner would legitimately see.

## Why this is not the previously-closed email root

An earlier submission on this account concerned a *different* decision in a neighbouring file — an unauthenticated verification route resolving a channel by token (the "pre-claim" root). This report is about the **limiter's key**: the counter is bound to the address instead of the requester, so the defect lives in `email-verification-rate-limit.ts`, and its consequence is a lockout, not a channel claim. The files, the decision and the effect are all different, and fixing the pre-claim root would not fix this one.

## Impact Details

* **A victim is temporarily denied a feature of the target application by anyone who knows their email address**: their own verification request is refused for at least an hour, renewable without limit, so a determined attacker can keep it refused indefinitely.
* **No mail reaches the victim** during the window, so the victim cannot complete verification by retrying.
* The attacker needs nothing beyond an ordinary account: no wallet interaction by the victim, no cooperation, no access to their inbox.
* Side effect visible in the same run: the attacker's requests each cause a real verification email to be sent to the victim's address, so the victim also receives unsolicited mail authored by the attacker's actions.

Selected impact: *Temporarily disabling user to access target site, such as: Locking up the victim's funds, cookies or browser storage, or temporarily blocking the victim's IP address.*

## Honest limits

* The refusal is a delay, not damage: once the window lapses the victim verifies normally, and the state is not corrupted. A triager may therefore treat this as repairable rather than as the impact wording above, and may also argue that the limiter is intentionally shared as anti-spam protection. Those are the two arguments this report has to survive, and they are stated here rather than hidden.
* The harness drives the real route modules in process with the KV, database handle, RPC client and outbound mail boundaries substituted; the substitution is documented in the run log.

## Suggested fix

Key the limiter on the requesting principal together with the address — for example `notifications:email-verification:<user_id>:<email>` — or require the caller to prove control of the address before the counter is consumed.

## Link to Proof of Concept

<https://gist.github.com/crypt0b0y/6309c88665f894e374479fd2582834bb>

## Proof of Concept

## Environment

* Frozen repository revision: `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* Local only. The worker is driven **in process** — no deployed host is contacted, and no mail is sent to a real address (the SendGrid send path is replaced by a counting stub and the recipient is a reserved example address).
* No existing repository file was modified; everything is additive under `apps/portal/audit-artifacts/medium-low-hunt-20260912/`.

## How to reproduce

```powershell
cd D:\Pentest\bb\ens-competition\workers\api-worker
.\node_modules\.bin\vitest.CMD run ..\..\apps\portal\audit-artifacts\medium-low-hunt-20260912\faucet-and-rate-limit.probe.test.ts
```

Suite result: 8 of 8 cases pass. Raw log: `run-worker-probes.log` in the same directory.

## What is real and what is substituted

**Real**: the frozen worker's Hono application with its real routes, middleware, error handler and `requireJWT`/SIWE chain; the real rate-limiter module (`services/notifications/email-verification-rate-limit.ts`) with its real key construction and window logic; the real KV interface; the real channel-creation and verification handlers; the real Drizzle predicate trees (the fake store prints the SQL it would run).

**Substituted, at the binding boundary only**: the KV namespace implementation, the database handle, the RPC client, and the outbound SendGrid call (replaced by a stub that records the recipient and increments a counter). No route handler, limiter function, key constructor or predicate is re-implemented.

## The defect, in code

```ts
// workers/api-worker/src/services/notifications/email-verification-rate-limit.ts:29-30
const key = KV_KEY.NOTIFICATIONS.EMAIL_VERIFICATION(normalizeEmailForRateLimit(email))
// -> notifications:email-verification:<email>

// consumed by the two mail-sending routes, neither of which passes a user id:
// app/routes/notifications/channels/email.ts:126
// app/routes/notifications/channels/$id.ts:173
```

`EMAIL_VERIFICATION_RATE_LIMIT_MAX_SENDS = 3` per window; the window is one hour.

## Observed output (verbatim excerpts)

```
[B1] key written = ["notifications:email-verification:victim.person@example.com" ×3, …someone.else…]
[B1] victim-facing string = Too many verification emails sent to this address. Try again in 60 minutes.

[B2] attacker request 1 -> 200  mail -> victim.person@example.com  (send count 1)
[B2] attacker request 2 -> 200  mail -> victim.person@example.com  (send count 2)
[B2] attacker request 3 -> 200  mail -> victim.person@example.com  (send count 3)
[B2] victim   request   -> 429  {"error":"Too many verification emails sent to this address. Try again in 60 minutes."}
[B2] send count after the victim's own request: 3  (i.e. 0 mails for the victim)

[B3] after the hour window lapses (clock advanced in KV state):
     victim request -> 200, one mail to victim.person@example.com
```

So the refusal is bound to the address, is renewable by the attacker at three requests per hour, and lapses only with the clock.

## Controls in the same suite

* A request from a **different** address under the same account is unaffected while the victim's key is exhausted (the limiter is per address, not per account), which is what shows the key is the address.
* The SQL the store would run for the channel queries is printed and carries the JWT `user_id` (`user_channels.user_id = $1 and …`), so the route itself is correctly scoped — the defect is the limiter key, not an authorisation gap.

## Limitations, stated plainly

* The refusal is temporary and self-healing after the window; the attacker can renew it, but nothing is corrupted and nothing is stolen.
* KV state is modelled at the binding boundary, so the exact expiry semantics depend on the binding's implementation (`expirationTtl`), which the harness reproduces from the worker's own call arguments.
* No real email is sent anywhere in the reproduction.
