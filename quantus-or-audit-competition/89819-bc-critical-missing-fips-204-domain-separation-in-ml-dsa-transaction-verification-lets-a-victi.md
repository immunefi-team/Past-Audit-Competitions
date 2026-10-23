# 89819 bc critical missing fips 204 domain separation in ml dsa transaction verification lets a victim s message signature authorize a transfer enabling direct theft of user funds

**Submitted on Aug 21st 2026 at 10:54:03 UTC by @zealynx for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89819
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The Quantus runtime verifies every transaction's ML-DSA signature under the empty FIPS 204 context (`ctx = None`) — the same context its from-scratch signing scheme uses for generic message signing. FIPS 204 provides `ctx` specifically to separate signing domains; Quantus leaves it unused on both the signing and the verifying path, so at the cryptographic level a signature over an arbitrary message is indistinguishable from a transaction authorization. A signature a victim produces over any byte string that happens to be a valid SCALE-encoded `SignedPayload` is, byte-for-byte, a valid transaction signature — so an attacker who obtains one such signature moves the victim's funds. The proof of concept transfers 100 QUAN from the victim to the attacker and advances the victim's nonce, with no access to the victim's key.

## Vulnerability Details

Quantus implements ML-DSA (Dilithium) from scratch and integrates it as the runtime signature scheme. That integration discards the standard's domain-separation mechanism on both paths:

* The `Pair` implementation signs every input with no context — `secret.sign(message, None, None)` (`primitives/dilithium-crypto/src/scheme_macro.rs:365`).
* The runtime's transaction-signature verifier checks every `SignedPayload` with no context — `pk.verify(msg, sig, None)` (`primitives/dilithium-crypto/src/scheme_macro.rs:495`), reached through the runtime `Verify for DilithiumSignatureScheme` adapter (`primitives/dilithium-crypto/src/traits.rs:152-183`).

FIPS 204 defines `ctx` as a first-class, standardized domain-separation input for exactly this situation: a signature made in one context must not verify in another. This is the crux, and it is what separates this from ordinary Substrate signing — **sr25519 and ed25519 have no such primitive** and therefore depend on clients wrapping user messages before signing; **ML-DSA carries `ctx` in the standard itself**, and Quantus's integration leaves it empty on both paths. As a result the runtime has no way to tell a signature intended as a message acknowledgement from one intended as transaction authorization.

The account-to-public-key binding does not prevent this: a raw-message signature contains the victim's real public key, so the binding (`account == hash(embedded_pubkey)`) succeeds *before* the context-free ML-DSA verification succeeds.

The same `Pair::sign` is the single signing primitive behind both transaction construction and generic message signing — the SDK's `signMessage` and the node's `sign` subcommand both route through it. A victim performing an ordinary, expected action — signing a login or ownership challenge — therefore produces a signature that is simultaneously a valid transaction authorization.

### Vulnerable scenario

{% stepper %}
{% step %}

## Construct the transaction payload

The attacker builds a transfer from the victim to the attacker with the correct call, nonce, era, fee, spec/transaction version, and genesis, and SCALE-encodes the resulting `SignedPayload` (a 119-byte string, below Substrate's 256-byte hashing threshold, so it is signed as-is).
{% endstep %}

{% step %}

## Obtain a generic message signature

The attacker presents that opaque byte string to the victim to sign through the generic ML-DSA message-signing operation — e.g. as a login/ownership challenge. The victim authorizes a *message*, is shown no transaction, and never discloses key material.
{% endstep %}

{% step %}

## Submit the signed extrinsic

The attacker inserts the returned signature, unchanged, into the prebuilt extrinsic and submits it through the public transaction interface.
{% endstep %}

{% step %}

## Runtime execution

The runtime verifies the signature under `ctx = None`, attributes the extrinsic to the victim, dispatches the transfer, charges the victim's fee, and increments the victim's nonce.
{% endstep %}
{% endstepper %}

### Recommended fix

Bind a fixed transaction domain into verification, at the Quantus integration layer — this is a minimal runtime/crypto change, not client-side message wrapping:

```diff
// primitives/dilithium-crypto/src/scheme_macro.rs  (runtime transaction verify)
-            Ok(pk) => pk.verify(msg, sig, None),
+            Ok(pk) => pk.verify(msg, sig, Some(b"quantus/extrinsic/v1")),
```

Have transaction construction sign with the same context, and keep the generic message-signing API on a distinct context (or `None`). A signature produced for a message can then never satisfy the transaction verifier.

## Impact Details

Direct theft of user funds. An attacker who obtains a single signature from the victim's generic message-signing operation executes the transaction encoded in that message as the victim — a transfer of the victim's balance to the attacker, with the victim charged the fee and the victim's nonce advanced. The attacker never holds the victim's secret key; one signature over attacker-chosen bytes suffices, and the bytes are a legitimate transaction the victim never saw as such. The proof of concept demonstrates an exact `100 QUAN` transfer to the attacker.

Selected in-scope impact: **Direct theft of any user funds, whether at-rest or in-motion** — funds move from the victim's account to the attacker's via a transaction the victim never authorized as a transaction.

## References

Frozen revision: `quantus-chain @ 3b243b870a5442a6e5f443056f67bbba87d472f5`

* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/scheme\\_macro.rs#L358-L370> — generic signing with `ctx = None`
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/scheme\\_macro.rs#L489-L500> — transaction verify with `ctx = None`
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L152-L183> — runtime `Verify` adapter

## Link to Proof of Concept

<https://gist.github.com/vendrell46/35dfead57d895aa6ecf8642c40102a89>

## Proof of Concept

Runs locally against commit `3b243b8`. **No test was run against mainnet or any public testnet.** The finding is demonstrated by two executed tests of the same root cause, one per shipped ML-DSA parameter set. Each models the honest key-holder as an oracle exposing only the generic `Pair::sign` operation over an attacker-chosen message (the attack code never receives the secret), then applies a real `Balances::transfer_keep_alive` through `Executive::apply_extrinsic` with the full production `TxExtension`. Both test files are in the secret gist linked at the end.

### Step by step

{% stepper %}
{% step %}

## Construct and sign the payload

The attacker constructs the victim→attacker transfer `SignedPayload` and hands the raw bytes to the oracle's message-signing operation (`Pair::sign`), receiving `Dilithium{65,87}SignatureWithPublic`.
{% endstep %}

{% step %}

## Insert and apply the signature

The attacker inserts that signature unchanged into the extrinsic and applies it. The runtime accepts it, transfers the funds to the attacker, and advances the victim's nonce.
{% endstep %}

{% step %}

## Causal control

The identical payload signed with a non-empty FIPS 204 context (`quantus:raw-message:v1`) is rejected by today's `ctx = None` runtime verifier, changing no balance or nonce — proving the context is the load-bearing separation.
{% endstep %}

{% step %}

## Defect mutation

Changing only the verify line to `Some(b"quantus:transaction:v1")` makes the unchanged exploit fail with `InvalidTransaction::BadProof`; reverting restores the theft — pinning the harm to the cited line.
{% endstep %}
{% endstepper %}

### Run

{% code overflow="wrap" %}

```sh
# Dilithium65
SKIP_WASM_BUILD=1 cargo test -p quantus-runtime --test mod \
  raw_message_signature_replays_as_victim_transfer -- --nocapture
# Dilithium87 (primary scheme)
SKIP_WASM_BUILD=1 cargo test -p quantus-runtime --test mod \
  dilithium87_auth_challenge_signature_forges_victim_transfer -- --nocapture
```

{% endcode %}

### Observed output

```
CHALLENGE_BYTES=119
UNAUTHORIZED_TRANSFER=100000000000000
VICTIM_DEBIT=100012953787000
VICTIM_NONCE=1
test ... ok
```

The victim is debited the transfer plus fee, the attacker is credited exactly the transfer, and the victim's nonce advances to 1 — a transaction executed as the victim from a signature the victim produced for a message.

**Full PoC files (Dilithium65 + Dilithium87 tests):** <https://gist.github.com/vendrell46/35dfead57d895aa6ecf8642c40102a89>
