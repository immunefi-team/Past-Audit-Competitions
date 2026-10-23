# 92979 w a insight telegram authentication accepts future dated auth date values

**Submitted on Sep 13th 2026 at 16:00:19 UTC by @blackgrease for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92979
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**

## Description

The ENS Telegram notification endpoint has a validation flaw when enforcing `auth_date`. The verification process checks that the timestamps is not too old but fails to check if the timestamp is in the future violating a security best practice of rejecting authentication data containing a future-dated timestamp.

### Root Cause

In `/workers/api-worker/src/services/telegram/auth.ts`, the verification logic currently does

```ts
//--snip--
if (Date.now() - data.auth_date * 1000 > maxAge) {
    return error({
        code: 'TELEGRAM_AUTH_DATA_EXPIRED',
        message: 'Auth data expired',
    })
}
//--snip--
```

which checks that the authenticated data has been signed recently, i.e within `maxAge`.

However, the logic fails to explicitly validate that the `auth_date` is not in the future i.e `auth_date` > current time.

Therefore, when a `auth_date` is a future timestamp, the ENS verification effectively calculates `current time - auth_date`. Because the result is negative, the comparison against `maxAge` evaluates to `false`, causing the age check to pass.

Essentially the logic is *only* checking "Is this authentication too old?" without also checking "Is this authentication from the future?" violating a security best practice in signature validation processes regarding time.

**Example scenario**

Assume:

* Current time: 2026-09-13 16:00
* auth\_date: 2027-09-13 16:00

1. ENS logic calculates `current - auth_date = -365 days`
2. The logic checks if `-365 days > 1 hour?` which evaluates to `false`.
3. The timestamp passes the age check.

### Exploitability limitations

1. An attacker cannot take a legitimate Telegram response and change the `auth_date` to a value of their choosing as this will invalidate the signed hash. This brings us to the second limitation.
2. Exploitation would require access to the ENS Telegram bot token, or another means of obtaining a correctly signed authentication payload containing a future `auth_date`.
3. It also requires access to a victims authenticated ENS account

## Impact

The purpose of `auth_date` is specifically to constrain the lifetime of an authentication assertion. By failing to enforce both time-related directions (past and future), ENS accepts signed authentication data outside the intended freshness interval. A future-dated assertion can therefore pass validation despite not representing an authentication event that has occurred yet.

This issue violates standard security best practices in cryptography where data containing a future `auth_date` timestamp should not be accepted due to the risks that it introduces e.g replay attacks.

As per Telegrams documentation wording, the validation of the `auth_date` field is left to the developers.

> To prevent the use of outdated data, you can additionally check the auth\_date field, which contains a Unix timestamp when the authentication was received. - <https://core.telegram.org/widgets/login/?utm\\_source=chatgpt.com#checking-authorization>

As there are considerable limitations in the practicality of exploiting, this issue falls under "Security best practices" as it represents a real issue not currently exploitable but an area for improvement.

## Recommendations

Consider enforcing both sides of `auth_date` - both past and future.

Example implementation of fix in `workers/api-worker/src/services/telegram/auth.ts`:

```diff
//snip--
+   const age = Date.now() - data.auth_date * 1000

-   if (Date.now() - data.auth_date * 1000 > maxAge) {
-      return error({
-      code: 'TELEGRAM_AUTH_DATA_EXPIRED',
-      message: 'Auth data expired',
-    })
-  }

+   if (age < 0) {
+     return error({
+        code: 'TELEGRAM_AUTH_DATA_INVALID_TIMESTAMP',
+        message: 'Auth data timestamp is in the future',
+    })
+    }

+   if (age > maxAge) {
+     return error({
+        code: 'TELEGRAM_AUTH_DATA_EXPIRED',
+        message: 'Auth data expired',
+   })
+   }
```

## Proof of Concept

The following test demonstrates that `verifyTelegramAuth()` accepts a correctly signed authentication payload whose `auth_date` is one year in the future.

The PoC uses a test bot token to generate a correctly signed future-dated authentication payload. This demonstrates the verifier's acceptance of the invalid timestamp; it does not claim that an external attacker can forge Telegram authentication without access to the bot's signing secret.

1. Place the test script in: `workers/api-worker/src/services/telegram/poc_telegram_authdate_validation_flaw.test.ts`
2. Run with: `pnpm test src/services/telegram/poc_telegram_authdate_validation_flaw.test.ts`

### Console output

```bash
 ✓ src/services/telegram/poc_telegram_authdate_validation_flaw.test.ts (2 tests) 7ms
   ✓ verifyTelegramAuth (2)
     ✓ test 1: accepts a valid current Telegram auth timestamp 4ms
     ✓ test 2: accepts an auth_date one year in the future 0ms

 Test Files  1 passed (1)
      Tests  2 passed (2)
   Start at  18:00:13
   Duration  1.59s (transform 110ms, setup 0ms, import 226ms, tests 7ms, environment 0ms)
```

### The Code

```ts
import { describe, expect, it } from 'vitest'
import type { TelegramAuthData } from '@ens-apps/shared-schema/telegram'
import { verifyTelegramAuth } from './auth.js'

const TEST_BOT_TOKEN = '123456789:test-bot-token'

async function createTelegramHash(
  data: Omit<TelegramAuthData, 'hash'>,
  botToken: string,
): Promise<string> {
  const dataCheckString = Object.keys(data)
    .sort()
    .map((key) => `${key}=${data[key as keyof typeof data]}`)
    .join('\n')

  // Telegram derives the HMAC secret by SHA-256 hashing the bot token.
  const botTokenBytes = new TextEncoder().encode(botToken)

  const secret = await crypto.subtle.digest(
    'SHA-256',
    botTokenBytes,
  )

  const cryptoKey = await crypto.subtle.importKey(
    'raw',
    secret,
    {
      name: 'HMAC',
      hash: 'SHA-256',
    },
    false,
    ['sign'],
  )

  const checkStringBytes = new TextEncoder().encode(dataCheckString)

  const signature = await crypto.subtle.sign(
    'HMAC',
    cryptoKey,
    checkStringBytes,
  )

  return Array.from(new Uint8Array(signature))
    .map((byte) => byte.toString(16).padStart(2, '0'))
    .join('')
}

describe('verifyTelegramAuth', () => {
  it('test 1: accepts a valid current Telegram auth timestamp', async () => {
    const authDataWithoutHash = {
      id: 123456789,
      first_name: 'Test',
      username: 'testuser',
      auth_date: Math.floor(Date.now() / 1000),
    }

    const hash = await createTelegramHash(
      authDataWithoutHash,
      TEST_BOT_TOKEN,
    )

    const authData: TelegramAuthData = {
      ...authDataWithoutHash,
      hash,
    }

    const result = await verifyTelegramAuth(
      TEST_BOT_TOKEN,
      authData,
    )

    expect(result.isOk()).toBe(true)
  })

  it('test 2: accepts an auth_date one year in the future', async () => {
    const now = Math.floor(Date.now() / 1000)

    // One year in the future.
    const futureAuthDate = now + 365 * 24 * 60 * 60

    const authDataWithoutHash = {
      id: 123456789,
      first_name: 'Test',
      username: 'testuser',
      auth_date: futureAuthDate,
    }

    // Generate a valid Telegram-style signature for the FUTURE
    // timestamp so that the test reaches the timestamp validation.
    const hash = await createTelegramHash(
      authDataWithoutHash,
      TEST_BOT_TOKEN,
    )

    const authData: TelegramAuthData = {
      ...authDataWithoutHash,
      hash,
    }

    const result = await verifyTelegramAuth(
      TEST_BOT_TOKEN,
      authData,
    )

    expect(result.isOk()).toBe(true)
  })
})

```
