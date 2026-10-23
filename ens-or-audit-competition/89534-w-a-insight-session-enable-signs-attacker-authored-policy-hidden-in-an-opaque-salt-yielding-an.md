# 89534 w a insight session enable signs attacker authored policy hidden in an opaque salt yielding an effectively permanent attacker keyed session

**Submitted on Aug 19th 2026 at 18:49:27 UTC by @Griffinf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89534
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/smart-account>
* **Impacts:**
  * Unauthorized takeover of a user's smart account (Hybrid Custody Account) or authenticated session, letting an attacker transfer the victim's names or funds or act as them, such as:
    * adding, replacing, or escalating an account owner or co-owner
    * extending or widening a session key's authority beyond its intended scope or lifetime
    * stealing an authenticated session or bearer token

> Exploiting the 7-day session-key design as described in the Known Issues is not eligible; a weakness that grants authority beyond its stated lifetime or permissions is in scope.

## Description

**Severity claimed: Critical.**

This maps verbatim to the in-scope Critical impact "Unauthorized takeover of a user's smart account (Hybrid Custody Account) or authenticated session, letting an attacker transfer the victim's names or funds or act as them", whose examples include "extending or widening a session key's authority beyond its intended scope or lifetime" and whose closing parenthetical states "a weakness that grants authority beyond its stated lifetime or permissions is in scope". That is what I demonstrate, with exact numbers: the stated session lifetime is 24 hours and the stated gas-refund ceiling is 100 USDC; the attacker-authored enable, which no readable field distinguishes from an honest one, carries `validUntil = 2^48-1` (roughly year 8.9 million) and `maxRefundAmount = 2^96-1`, under a session key the attacker holds and the victim's machine never stores.

**Precondition, stated plainly:** one moment of script execution in the Manager origin at the time the user enables a session (any injection foothold, or an equivalent one-time storage-write moment). I claim nothing about the delivery vector; the Known Issues disclosure itself partitions foothold from escalation ("an actual injection sink is a separate finding", from the entry on the absent CSP), prices an assumed storage-write attacker at High elsewhere (the base-URL override entry), and lists localStorage modification in the paid impact list. Independent of any foothold, this is the signing-correctness class the program overview prioritizes: a transaction signed with arguments the signer cannot see.

**The expected pushback, answered:** same-origin script can always ask the connected wallet to sign something; that is not the finding. The escalation is that the owner signature does not bind any human-readable security parameter: the protocol reduces the session's lifetime, caps, and key to opaque salt and init-data bytes, so the wallet cannot distinguish a legitimate 24-hour session from an effectively permanent attacker session, and the deployed validator then enforces the hidden policy exactly as authored (Proof of Concept: the verbatim on-chain SessionConfig readback, the honest control already expired at +25 hours while the attacker session still repoints records and pulls refunds, acceptance of a fresh attacker intent at +365 days).

**Asset:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/smart-account> plus `apps/manager` (session actors) and the repository's modified `@rhinestone/sdk@1.8.0` patch (modified third-party code is in scope per the program's scope boundary statement). Verified at the `audit-comp-ready` branch head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`; the program page lists audited revision `63772fd872af472ced58b009499355f3430c2a86`; per the repository's own history it is the fork point (commit `cda79ac`), and the only commits between it and the verified head are a hono dependency bump and its merge, touching none of the cited paths.

## Relationship to published Known Issues

* **R2-03** is the nearest entry, and it describes the retired architecture: an unscoped session key that is a "time-boxed co-owner" for 7 days, on an account that "locks its module set and rejects scoped-session modules", exfiltratable from browser storage. The audited revision does not ship that model. The header of `packages/smart-account/src/providers/rhinestone/session.ts` states the session is "a scoped ERC-7579 SmartSession on the standalone `HCAOwnerAndSessionValidator`" and explicitly "NOT the old ephemeral-owner model"; the default stated lifetime is 24 hours (`DEFAULT_SESSION_VALIDITY_SECONDS`, `manifest.ts` line 189). My root cause is the **enable protocol of the new model** (what the owner's one signature commits to), not key custody or key breadth; no entry describes the enable path, the salt, request-side policy authoring, or `getSessionDetails`. R2-03's own carve-out names this report's exact shape, verbatim: "A weakness that grants authority **beyond** the session's stated lifetime or the account's own permissions would be a new finding."
* **R2-06** ("Session revocation is local-only") compounds the severity rather than fencing the finding: `StandaloneSingleOwnerHCA.revokeSessions()` exists on-chain but nothing in the app calls it and no UI surfaces it, so the authored forever-session has no practical bound. I cite it as the reason the damage is permanent, not as a finding.
* **SEC-TXM-002** (missing cross-check of the cached smart-account address, "no concrete attack path") is a different root cause; the account rebinding I demonstrate rides request-side data in the patched SDK's enable path, not the cached-address read.
* **EXP-INPUT-003** fences locally writable persisted state "primarily as a denial of service"; authoring session policy through the signing protocol is neither DoS nor a shape-validation issue.
* No entry mentions the session salt, `computeDestinationSessionSalt`, `SessionInput.salt`, `SessionInput.account`, or any bound on `validUntil`. This includes the eight R3 entries added in the 14 August 2026 update (R3-01 through R3-08), which cover telemetry, transaction-actor liveness, retry sequencing, and history/archive bookkeeping; none of them describes the session enable path, the salt, or any session policy field.

This report is related to, but has a distinct root cause and fix from, my report "transaction-manager constructs, session-signs and submits attacker-supplied custom intents with zero validation and no consent surface: prompt-free repointing of the victim's registered names plus repeatable refund burn from one script-execution moment". That report is about the runtime absence of any app-layer gate on what an already-live session signs (root cause in `packages/transaction-manager`; fix is intent validation and consent). This one is about the enable-time authoring protocol: what the owner's single wallet signature commits to (root causes in `packages/smart-account`, `apps/manager`, and the SDK patch; fix is typed policy fields and salt verification). Fixing either leaves the other fully exploitable.

## Description

**Root cause, stated up front:** the session-enable protocol lets same-origin code obtain an owner signature over attacker-selected session policy while hiding the security-critical policy values behind opaque salt/init-data fields. The application-level defaults do not constitute signed bounds.

Enabling a session takes exactly one wallet prompt: the "enable session" signature over a `MultiChainSession` EIP-712 payload. That payload's only carrier of the actual session policy is `salt`, an opaque `bytes32` that is `keccak256` of the real policy scalars, computed **in page context**. Every policy-bearing human-readable field of the signed struct is constant for every enable: `expires` is always `maxUint256` (a stock SDK constant the patch retains), the `actions` array is a sudo placeholder (selector `0x00000001` at target `0x...01`), and the session key appears only as raw bytes inside `sessionValidatorInitData`; the readable fields that do vary (`account`, chainId) carry no policy. An honest 24-hour enable and a malicious forever-enable for the same account therefore differ only in opaque bytes (the salt hash and the raw init data); no wallet can display policy fields that are not in the typed data.

This opacity is not inherited from stock Rhinestone; it is the in-scope delta. Stock expresses session policy as typed, wallet-renderable `actions` and hardcodes the salt to `zeroHash`. The repository's patch introduces `SessionInput.salt` and `SessionInput.account` as request-side fields and threads them into the signed struct unverified, and the in-scope application code relocates the entire effective policy (validUntil, resolver, refund token, all three refund caps) into that salt. The deployed on-chain validator confirms the design is structural: its digest reconstruction hardcodes `expires` to `type(uint256).max` and `permissions` to the empty-permissions hash, so the signed struct has no channel that could carry a readable expiry or cap. The patch does add a salt cross-check on the funding path (`resolveHcaFundingSessionConfig`), so the codebase already treats request-side salts as requiring verification; the destination enable path has none.

Script present at the enable moment does not need the app's session actor at all. `createSessionActor`'s public config exposes only an optional `validUntil` (the session key is generated inside the actor; the resolver and refund constants are app-internal), so the attack bypasses the actor and drives the lower-level primitives it is built on: the in-scope salt helper to hash attacker policy, and the patched SDK's request-side `session.salt` and `session.account`, which `getSessionDetails` uses when constructing the signed session. The substituted inputs: attacker session key (generated off-device, never stored on the victim machine), `validUntil = 2^48-1`, `maxRefundAmount = 2^96-1`, salt computed over that policy. The owner signs the same-looking prompt. On-chain, the deployed `HCAOwnerAndSessionValidator`'s enable-proof check accepts on the session nonce, `validUntil >= now`, the refund-token whitelist, and the owner's ECDSA over this very digest; the enable proof is reusable and never consumed. Because `session.account` is also request-side, the same blind signature can be pre-bound to any victim-owned account, including the deterministic future account address (shown at the signed-struct level by the offline proof). The only revocation is the owner calling `revokeSessions()` directly, which the app never does.

## Vulnerability Details

Paths relative to the repository root at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

### The sanctioned actor surface, for contrast: one exposed policy field, and even it is unbounded

`apps/manager/src/lib/smart-account/actors/session.actors.ts` lines 84 to 96:

```ts
export function createSessionActor(
  input: CreateSessionInput,
): ResultAsync<CreateSessionOutput, SessionEnableError> {
  const sessionPrivateKey = generatePrivateKey()
  ...
  const validUntil = BigInt(
    input.config?.validUntil ??
      Math.floor(Date.now() / 1000) + DEFAULT_SESSION_VALIDITY_SECONDS,
  )
```

`config.validUntil` is accepted with no upper bound; the 24-hour figure is only a default. This is the only policy field the actor's public config exposes (`readonly config?: { readonly validUntil?: number }`; the key is generated inside the actor, and the resolver and refund constants are app-internal), which is the honest measure of the application-level defaults: they are defaults, not signed bounds. The full attack does not go through the actor at all; it drives the two lower-level channels documented next.

### The whole policy folds into one opaque salt

`packages/smart-account/src/providers/rhinestone/session.ts` lines 107 to 138: `computeDestinationSessionSalt` is `keccak256(abi.encode(nonce, validUntil, resolver, refundToken, maxRefundExchangeRate, maxRefundGasOverhead, maxRefundAmount))`. Those are all of the session's real constraints, and this hash is the only place they appear in what the owner signs.

### The patch makes the salt and the account request-side

`patches/@rhinestone%2Fsdk@1.8.0.patch`: `SessionInput.account?: Address; salt?: Hex` are patch-introduced (patch lines 1073 to 1075), `salt: session.salt ?? viem_1.zeroHash` replaces stock's hardcoded `salt: viem_1.zeroHash` (patch lines 1022 to 1023, in the compiled dist the patch targets), and `getSessionDetails` resolves the account it signs for from request-side `session.account ?? account` in both its nonce read and its signed-session construction (patch lines 995 and 998). These are the fields the attack hands its values to. The patch contains no `expires` hunk anywhere: `expires = maxUint256` is stock SDK code, a concession I state openly. The patch's funding-path salt cross-check (lines 636 to 657) guards only `resolveHcaFundingSessionConfig`, not the destination enable path exercised here.

### Stated policy for contrast

`packages/smart-account/src/providers/rhinestone/manifest.ts`: `DEFAULT_SESSION_VALIDITY_SECONDS = 24 * 60 * 60` (line 189) and `MAX_REFUND_AMOUNT = 100_000_000` with the comment "100 USDC ceiling" (line 205).

### What the wallet receives

My proof of concept prints the verbatim EIP-712 message handed to the wallet for signing (presentation varies by wallet; this payload is everything a wallet has to render): `expires` = 115792089237316195423570985008687907853269984665640564039457584007913129639935, `actions` = the sudo placeholder, the policy present only as the opaque `salt`, and the session key only inside raw `sessionValidatorInitData` bytes. The honest and malicious payloads differ only in the opaque salt hash and the raw init-data bytes; no field a human can interpret separates them.

## Impact

One script-execution moment at enable time converts to an effectively permanent, attacker-keyed capability on the victim's smart account:

{% stepper %}
{% step %}

### Record repointing of the victim's app-registered names

The attacker monetization. The session is bound to the victim's resolver, where the account holds root ALL\_ROLES, and the on-chain session policy accepts `setAddr`, `setText`, and `setContenthash` with node and value free, so it reaches every name whose records live on that resolver (app registrations write their records there). A third party who pays a repointed name pays the attacker.

Demonstrated on the fork at +25 hours and again past +7 days, and accepted at +365 days, while the honest control session was already expired at +25 hours.
{% endstep %}

{% step %}

### Repeatable gas-refund burn to the fixed paymaster

Value destruction, not attacker revenue. The authored `2^96-1` cap replaces the stated 100 USDC ceiling; my run pulled 150 USDC twice (cumulative 300 USDC), each pull above the stated per-intent cap.
{% endstep %}

{% step %}

### Registration burn

Unlimited `approve` to the registrar with free duration, label, and subregistry within policy.
{% endstep %}

{% step %}

### Pre-binding

`session.account` being request-side, the same blind signature can be pre-bound to any victim-owned account, including the deterministic future account address of a user who has not yet deployed one (shown at the signed-struct level by the offline proof; not exercised on-chain).
{% endstep %}
{% endstepper %}

Persistence amplifier, not a standalone claim: the app persists the session record to `localStorage["ens-sessions-v9"]` in cleartext, including the session private key and the owner's reusable enable signature, and the patch keeps that enable payload attached on standalone-account transactions. Any transient read of that value extends the exposure for the whole authored lifetime.

## Link to Proof of Concept

<https://gist.github.com/aussinfosec/7d0333009358909c438873f1ac8c2730>

## Proof of Concept

Two scripts, both run from a clean checkout at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` with `pnpm install`, both local-only with zero production contact (the fork's only upstream traffic is read-only `eth_*` against the repo's own pinned Sepolia RPC).

A self-contained reproduction bundle (both scripts and the full run log) is available at <https://gist.github.com/aussinfosec/7d0333009358909c438873f1ac8c2730> (secret gist).

{% stepper %}
{% step %}

### Offline authoring proof, `packages/transaction-manager/poc-chain-a/poc-b-forever-session.ts`

Run:

```bash
e2e/node_modules/.bin/tsx packages/transaction-manager/poc-chain-a/poc-b-forever-session.ts
```

Drives the real patched-SDK enable path (`getSessionDetails` plus owner `signTypedData`; the single nonce read is served by a local stub with the same undeployed-account value the app uses). Six assertions pass: the real expiry exists only inside the salt; the refund caps exist only inside the salt; the struct's readable expiry is always `maxUint256`, so no readable field separates an honest enable from a malicious one; the session key appears only as raw init-data bytes; `session.account` is request-side and rebinds the same signature to an arbitrary account; and the salt preimage recomputes exactly. The script prints the verbatim wallet-visible EIP-712 message quoted above.
{% endstep %}

{% step %}

### End-to-end on-fork proof against the deployed validator, `packages/transaction-manager/poc-chain-a/e2e-1-forever-session-onchain.ts`

Start a fork:

```bash
anvil --fork-url <the SEPOLIA_FORK_URL pinned in e2e/infra/docker-compose.yml> --chain-id 11155111 --port 8556
```

Then run:

```bash
e2e/node_modules/.bin/tsx packages/transaction-manager/poc-chain-a/e2e-1-forever-session-onchain.ts
```

My full run log is attached as `e2e-1-run.log` (fork base: Sepolia block 11515580).

Stages, all passing:

* A fresh victim owner (`0x2271Fe8BF94E01007Cdb382704156b74D55092fB`) deploys its account through the real factory at the address the in-scope helper computes (`0xFaD69F6bf4dd707298Af42c6D5C6Ad35445fA8A3`); the app's own `verifyStandaloneHca` accepts it.
* The attacker-authored enable is built through the real patched-SDK path with attacker session key `0x05075a3B0ebabE2816A292cD42F3A117961F0879` (never on the victim machine), `validUntil = 281474976710655` (2^48-1), `maxRefundAmount = 2^96-1`, and the attacker-computed salt. The victim owner signs once: the one blind prompt, whose payload `expires` field is `maxUint256`.
* **The gate:** `eth_call isValidSignatureWithSender(INTENT_EXECUTOR, digest, envelope)` on the DEPLOYED `HCAOwnerAndSessionValidator` at `0x5f249FCa8bB4949105651146858c347E8BFb0F7E` returns the ERC-1271 magic value `0x1626ba7e` for the attacker-authored mode-0x05 enable envelope, closing the digest-equivalence question against deployed bytecode. Negative controls: a tampered owner signature reverts `InvalidSigner` (`0x815e1d64`); a wrong sender reverts `CallerNotIntentExecutor` (`0x037b5679`). A bytecode drift gate first verified every load-bearing constant of the deployed runtime against the validator source.
* After the fill, the on-chain `SessionConfig` readback shows the authored policy stored **verbatim**: session key = attacker, `validUntil = 0xffffffffffff`, `maxRefundAmount = 2^96-1` (raw storage words and the `SessionEnabled` and `SessionRefundConfigured` events are in the log). The victim's own registration then rides the attacker-keyed session: it completes normally (the name registers to the victim and `addr(victimNode)` reads the victim owner), giving the victim no failure to notice.
* **Authority beyond every stated lifetime:** at +25 hours the honest 24-hour control session's identical intent reverts `SessionExpired` (`0x1fd05a4a`) while the attacker session repoints the victim's records on-chain (`addr(victimNode)` changed) and pulls a 150 USDC refund, above the stated 100 USDC cap. Past 7 days (at +7 days 2 hours) the record changes again with a second 150 USDC pull (cumulative 300 USDC to the fixed paymaster). At +365 days the deployed validator still accepts a fresh attacker intent.

**Substitutions, stated plainly:** the Rhinestone orchestrator (an out-of-scope remote service) is not in the loop; authorization was validated against the deployed on-chain validator from the exact production caller shape, and that on-chain check binds regardless of which relayer forwards the intent. Fills were sent as impersonated-account transactions, the same mechanism the repo's own pinned mock orchestrator uses; batch atomicity carries no authorization semantics. Untested: the production orchestrator's quoting and relaying policies, and mainnet deployment parameters (the program targets the Sepolia deployment the app ships against). My related report runs the companion attack through the live local orchestrator stand-in end to end.
{% endstep %}
{% endstepper %}

## Recommended Fix

{% stepper %}
{% step %}

### Bind the human-readable policy into the signed message

Bind the human-readable policy (expiry, refund caps, session key, resolver) into the signed EIP-712 message as typed fields so wallets render it, instead of a hashed salt; have the validator reconstruct from those fields.
{% endstep %}

{% step %}

### Verify and bound the policy

Until then: have the app refuse to sign enables whose salt it did not itself compute from app-fixed constants (the funding path already has exactly this cross-check), and enforce an upper bound on `validUntil` in `createSessionActor` and in the validator.
{% endstep %}

{% step %}

### Consume enable proofs

Consume enable proofs (single-use nonce) rather than leaving them reusable.
{% endstep %}

{% step %}

### Stop persisting the owner's enable signature

Stop persisting the owner's enable signature once the session is enabled on-chain, and encrypt the session record at rest.
{% endstep %}

{% step %}

### Surface session revocation

Surface `revokeSessions()` in the Manager UI so an over-authorized session has a practical remedy.
{% endstep %}
{% endstepper %}
