# 92837 w a insight safe owner removal does not revoke the ens api session they created

**Submitted on Sep 12th 2026 at 18:22:55 UTC by @PlasticGinnie98802 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92837
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes
  * Improperly disclosing confidential user information, such as: Email address

## Description

## Summary

ENS Manager accepts contract wallets through its normal WalletConnect and SIWE flow. The API then issues a JWT for the Safe address and accepts that token for seven days without checking whether the signer still controls the Safe.

That creates a stale-session problem when Safe ownership changes. Owner A can sign in while they control the Safe, receive a valid ENS JWT, and later be replaced by owner B. The Safe proxy address does not change. Fresh ERC-1271 signatures from A now fail, but A's existing JWT still authenticates as the same backend user.

I reproduced two consequences after the handoff:

1. Removed owner A used the old JWT to disable all three notification settings. B later read back the same persisted values with a fresh JWT.
2. A used the same JWT to read B's private notification email, which was added only after B took control.

Neither attacker request required any action from B. B's fresh-JWT read was a post-attack verification step.

I am reporting this as Critical. It matches the listed impact for taking authenticated notification or settings actions on behalf of another user without that user's interaction. The email disclosure is a secondary High impact from the same stale-session bug, not a separate report.

## Affected revision

* Competition import: `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* Competition branch: `audit-comp-ready@1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* Official upstream: `63772fd872af472ced58b009499355f3430c2a86`
* Imported tree: `8e0d79dac1ab4b4fdb80d6afed810087ae9f00ba`
* Safe implementation used by the proof: official Safe smart account v1.5.0 at `dc437e8fba8b4805d76bcbd1c668c9fd3d1e83be`

The affected Worker files are byte-identical between the competition branch and the official upstream material used by the formal evidence.

## Vulnerability details

### 1. The JWT outlives the authority that created it

`JWT_EXPIRATION` is seven days. `signJWT` places the address-associated payload and expiry in the token:

* `workers/api-worker/src/core/auth/jwt.ts:6,41-53`

Authenticated requests verify the HS256 signature and payload shape, then trust the embedded `address` and `user_id`:

* `workers/api-worker/src/app/middleware/auth.ts:7-35`

The middleware does not ask the wallet for another signature. It also does not run ERC-1271 or ERC-6492 again. A Safe address can keep existing after its owners change.

The backend stores one stable user row for that address:

* `workers/api-worker/src/core/database/schema/core.ts:8-15`

Replacing a Safe owner therefore changes the person with signing authority but not the backend principal carried in an already-issued JWT.

### 2. The stale JWT reaches state-changing notification routes

`PATCH /notifications/preferences` takes `user_id` from the authenticated JWT, upserts all three notification settings, and returns the persisted row:

* `workers/api-worker/src/app/routes/notifications/preferences.ts:62-120`

It does not check who currently controls the Safe. Removed owner A can keep writing settings for the stable Safe user until the JWT expires.

The earlier push-channel reproduction reaches the same boundary. `POST /notifications/channels/push` uses only the JWT-derived `user_id` and immediately inserts the supplied endpoint and key tuple with `status: 'verified'`:

* `workers/api-worker/src/app/routes/notifications/channels/push.ts:47-127`

### 3. The same token reads data added by the new owner

`GET /notifications/channels` selects channels for the JWT's `user_id`:

* `workers/api-worker/src/app/routes/notifications/channels/index.ts:30-52`
* `workers/api-worker/src/services/notifications/helpers.ts:29-43,99-114`

The response includes the email target as the channel label. In the proof, B's email was inserted only after the A-to-B owner replacement. A's pre-replacement token still returned it.

Notification creation later selects verified channels by the same `user_id`, copies their target into a delivery row, and queues delivery:

* `workers/api-worker/src/services/notifications/create.ts:115-218`

### 4. Fresh signatures correctly follow the new Safe owner set

The Safe side behaves as expected. `swapOwner` removes A and installs B while leaving the proxy address unchanged:

* `contracts/base/OwnerManager.sol:90-103` at Safe commit `dc437e8fba8b4805d76bcbd1c668c9fd3d1e83be`

The compatibility handler validates ERC-1271 signatures against the current owner set:

* `contracts/handler/CompatibilityFallbackHandler.sol:73-85` at the same Safe commit

After the handoff, a new signature from A fails and a new signature from B succeeds. The ENS JWT is the part that still treats A as authorized.

## Preconditions

1. A legitimately controls a Safe and signs into ENS once.
2. The Safe later removes or replaces A with B without changing the proxy address.
3. A keeps the JWT issued before removal and uses it before the seven-day expiry.

A does not need B's device, wallet, signature, browser storage, email account, or ENS name. Token theft is not required because A received the JWT legitimately while still authorized. The attack also needs no phishing or malicious Safe, and it does not assume that a contract wallet has an immutable controller.

## Impact

Once A has been removed, A is no longer a valid controller of the Safe. The old JWT nevertheless lets A act against the backend identity now controlled by B.

The reproduced state change is direct: A disables `ownedNameExpiry`, `favouritedNameExpiry`, and `ensLabsUpdates`. B performs no action during the attack and later reads all three values as `false`.

The same stale token also returns B's private notification email. That email did not exist when A controlled the Safe, so it is not data A was previously entitled to read.

The stale token remains usable for the rest of its seven-day lifetime. No provider failure or notification-delivery assumption is needed for the settings impact.

## Recommended fix

Do not treat control of a mutable contract-wallet address as permanent authority for the full JWT lifetime.

For security-sensitive authenticated writes, require a fresh wallet proof. Contract wallets should be checked again through ERC-1271 or ERC-6492 against current chain state. A shorter access-token lifetime would reduce the window, but it does not fix the ownership-epoch problem on its own.

A more complete design is to issue short-lived access tokens and require a fresh wallet proof when refreshing them. Sessions should also be tied to an authorization epoch that can change when contract-wallet control changes. Existing long-lived sessions should be revoked when this change is deployed.

## Duplicate assessment

This does not share the root cause of the submitted raw/canonical transfer or stale-manager migration reports, or the separate PostHog-toolbar candidate. It does not normalize an ENS name, read a stale indexer value, alter a transaction target, or depend on analytics behavior.

The earliest divergence is the API accepting a JWT created under an old ERC-1271 owner set after that owner set has changed.

The settings modification and email disclosure are two consequences of that one stale-session root. They should remain in one report.

## Verification

* Automated local reproduction: three tests, passed twice on fresh Anvil instances with byte-identical observations.
* The local harness reads the post-swap proxy independently from the mined transaction, checks that only B remains in `getOwners()`, and scopes the database double to the authenticated principal.
* Invalid bearer controls return `401` for both PATCH and GET.
* Independent review accepted the root as Critical and bounty-likely. The settings change and email disclosure remain one report.

## Proof of Concept

## Proof of Concept

This section gives the complete step-by-step reproduction method required by the audit-competition form. It is an explanatory PoC, not a standalone copy-paste test suite, and it does not rely on an external proof link.

I used the competition branch at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, Node.js 24.x, pnpm 10.27.0, and a fresh local Anvil chain configured with chain ID `11155111`. The chain hosted the official Safe v1.5.0 singleton, `SafeProxyFactory`, and `CompatibilityFallbackHandler`. The proof imported the unchanged ENS `signJWT`, authentication middleware, preferences route, channels route, and SIWE verification helper from the audited checkout.

The test boundary is explicit: I called `signJWT` directly after proving that A could authenticate the Safe, rather than automating the browser login endpoint. The real Hono routes ran in process. Their database middleware was replaced with a principal-aware in-memory double keyed by `c.var.user_id`, with an unrelated decoy user used to detect cross-principal reads or writes. B's verified email was added only after the owner replacement. This isolates the authorization failure while keeping the affected JWT and route code unchanged.

Reproduce the authorization sequence as follows:

1. Deploy and initialize a Safe whose only owner is A.
2. Create a SIWE message whose address is the Safe proxy. Sign the Safe message hash with A and pass it to the audited `safeVerifySiweMessage`. The result is valid.
3. For the backend user row associated with that Safe address, call the audited `signJWT({ address: safeProxy, user_id: stableUserId }, env)`. Keep the resulting token as `A_PRE_SWAP_JWT`.
4. Through the Safe itself, execute `swapOwner(0x0000000000000000000000000000000000000001, A, B)`.
5. Confirm independently that the mined `execTransaction` still targets the proxy emitted by `ProxyCreation`. Call `getOwners()` and confirm that the returned array contains only B.
6. Repeat the SIWE signature check using current chain state. A's fresh signature now fails, while B's fresh signature succeeds.
7. Only now add B's verified notification email, `new-owner-b-private-portable@example.invalid`, to the stable Safe user's database fixture.

With no request or signature from B, use A's pre-swap token for the following state-changing request:

```http
PATCH /notifications/preferences HTTP/1.1
Authorization: Bearer A_PRE_SWAP_JWT
Content-Type: application/json

{
  "ownedNameExpiry": false,
  "favouritedNameExpiry": false,
  "ensLabsUpdates": false
}
```

The affected route returns `200`. A subsequent `GET /notifications/preferences` using a fresh JWT created for B returns all three settings as `false`, proving that the stale session changed persisted state belonging to the post-handoff controller.

Use the same pre-swap token for the read consequence:

```http
GET /notifications/channels HTTP/1.1
Authorization: Bearer A_PRE_SWAP_JWT
```

The affected route returns `200` and includes `new-owner-b-private-portable@example.invalid`, even though that address was added only after A ceased to control the Safe.

As negative controls, repeat both requests with `Authorization: Bearer invalid-bearer`. Both return `401`. Also verify that an unrelated decoy user's notification settings and email remain unchanged and are never returned to the stable Safe user.

The automated reproduction produced the following observation:

```json
{
  "postSwapAValid": false,
  "postSwapBValid": true,
  "sameProxy": true,
  "stalePatchStatus": 200,
  "preferencesPersistedAllFalse": true,
  "settingsDisabledCount": 3,
  "victimInteractionCount": 0,
  "staleChannelsStatus": 200,
  "disclosedEmail": "new-owner-b-private-portable@example.invalid",
  "invalidPatchStatus": 401,
  "invalidChannelsStatus": 401
}
```

I also implemented this sequence in a local three-test harness and ran it twice against fresh Anvil instances. All three tests passed on both runs and produced byte-identical observations. The harness also asserted that `getOwners()` contained only B, the decoy user's settings remained unchanged, and neither user's email appeared in the other user's response.
