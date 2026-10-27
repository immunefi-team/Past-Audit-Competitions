# 88290 bc insight non canonical protobuf based peerid derivation in litep2p s noise handshake leads to reputation ban evasion in peerstore

**Submitted on Aug 12th 2026 at 15:57:29 UTC by @FaLLenSKiLL for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88290
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**

## Description

## Brief/Intro

litep2p — Quantus's post-quantum fork of the networking library that backs this node's peer-to-peer stack — derives a connecting peer's PeerId (the identity string trusted throughout the node, including the Peerstore reputation/ban table) by hashing the raw, sender-chosen byte encoding of that peer's public key, instead of a canonical form of the decoded key.

Because the wire format used for that encoding (protobuf) does not enforce a single canonical byte representation, a single Dilithium keypair can produce many different, equally valid PeerIds for the exact same identity. In production, this means a peer that has just been banned for misbehavior can reconnect immediately afterward using the same, still-banned key — re-encoded with one harmless extra byte — and be treated by the node as a brand-new, never-seen, unbanned peer.

## Vulnerability Details

{% stepper %}
{% step %}

## Wire format for the identity payload

```protobuf
client/litep2p/src/schema/noise.proto:
message NoiseHandshakePayload {
  optional bytes identity_key = 1;
  optional bytes identity_sig = 2;
  optional NoiseExtensions extensions = 4;
}

identity_key is itself the raw bytes of a nested, independently-encoded keys_proto::PublicKey message (client/litep2p/src/schema/keys.proto):
message PublicKey {
  required KeyType Type = 1;
  required bytes Data = 2;
}
```

{% endstep %}

{% step %}

## Two different consumers of the same raw bytes, two different semantics

```rust
client/litep2p/src/crypto/noise/mod.rs:721-747:
fn parse_and_verify_peer_id(
    payload: handshake_schema::NoiseHandshakePayload,
    kem_remote_pubkey: &[u8],
) -> Result<PeerId, NegotiationError> {
    let identity = payload.identity_key.ok_or(NegotiationError::PeerIdMissing)?;
    let remote_public_key = RemotePublicKey::from_protobuf_encoding(&identity)?;
    let remote_key_signature =
        payload.identity_sig.ok_or(NegotiationError::BadSignature)...?;

    let peer_id = PeerId::from_public_key_protobuf(&identity);

    if !remote_public_key
        .verify(&[STATIC_KEY_DOMAIN.as_bytes(), kem_remote_pubkey].concat(), &remote_key_signature)
    {
        ...
        return Err(NegotiationError::BadSignature);
    }

    Ok(peer_id)
}
```

Both `RemotePublicKey::from_protobuf_encoding` (used for signature verification) and `PeerId::from_public_key_protobuf` (used for the peer's identity going forward) are handed the exact same byte slice, `identity`. But:

* `RemotePublicKey::from_protobuf_encoding` (`client/litep2p/src/crypto/mod.rs:190`):

  ```rust
  pub fn from_protobuf_encoding(bytes: &[u8]) -> Result<RemotePublicKey, ParseError> {
      use prost::Message;
      let pubkey = keys_proto::PublicKey::decode(bytes)?;
      pubkey.try_into()
  }
  ```

  This is a stock `prost::Message::decode` call. prost implements standard protobuf wire-format decoding. Fields may appear in any order, unknown fields are skipped, and repeated occurrences of a non-repeated field overwrite the previous value with the last one seen. It is the protobuf specification's documented behavior. The function returns the semantic value of the message, discarding all information about how it was literally encoded on the wire.
* `PeerId::from_public_key_protobuf` (`client/litep2p/src/peer_id.rs:68`):

  ```rust
  pub fn from_public_key_protobuf(key_enc: &[u8]) -> PeerId {
      let multihash = if key_enc.len() <= MAX_INLINE_KEY_LENGTH {
          Multihash::wrap(MULTIHASH_IDENTITY_CODE, key_enc)...
      } else {
          Code::Sha2_256.digest(key_enc)
      };                                                                                                                                                                                                                                                                                          PeerId { multihash }
  }
  ```

  A Dilithium ML-DSA-87 public key, protobuf-wrapped, is far larger than `MAX_INLINE_KEY_LENGTH` (42 bytes), so this always takes tranch: the PeerId is SHA-256 of the literal bytes received on the wire.
  {% endstep %}

{% step %}

## Equivalent protobuf encodings produce different PeerIds

Because `Type` and `Data` are ordinary proto2 fields (not `oneof`/repeated), the following are all wire-valid re-encodings of the same `{Type: Dilithium, Data: <32xx bytes>}` value that prost decodes identically, per the protobuf specification's “last field wins” rule for duplicate singular fields:

* Canonical: `[Type][Data]`
* Reordered: `[Data][Type]`
* Duplicated Type: `[Type=bogus][Type=Dilithium][Data]`
* Duplicated Data: `[Data=bogus][Data=real][Type]`
* Padded varint: Type encoded with a non-minimal-length varint (e.g. `0x84 0x00` instead of `0x04` for value 4)
* Trailing unknown field: `[Type][Data][field 99: arbitrary bytes]`

All of these decode to the identical `keys_proto::PublicKey` struct via prost, so `RemotePublicKey::from_protobuf_encoding` returns any signature that verifies for one re-encoding verifies for all of them. The signature is computed and checked against the decoded key, `remote_public_key.verify(...)`, not against the raw bytes.

But each re-encoding is a different byte string, so `PeerId::from_public_key_protobuf` (SHA-256 of the raw bytes) produces a different PeerId for each one.

An attacker needs no cryptographic capability beyond what an ordinary, already-connected peer has: their own genuine Dilithium keypair and the ability to append a handful of extra bytes to the identity payload they send during the handshake.
{% endstep %}

{% step %}

## Where the resulting PeerId is trusted for a security decision

`client/network/src/litep2p/peerstore.rs`, Peerstore's reputation table is keyed by PeerId:

```rust
fn is_banned(&self, peer: &PeerId) -> bool {
    self.0.lock().peers.get(peer).map_or(false, |info| info.is_banned())
}

fn report_peer(&self, peer_id: PeerId, change: ReputationChange) {
    ...
    // Peer is currently banned, disconnect it from all protocols.
    ...
}
```

Because the whole Peerstore model assumes “one cryptographic identity ↔ one PeerId”, and that assumption is false for this network stack, a peer that has just been banned for misbehavior (invalid blocks, protocol violations, spam, etc. — whatever triggers a negative `ReputationChange` in this codebase) can dial back in immediately using the same Dilithium key, re-encoded with a single redundant protobuf field.

It authenticates successfully (same key, same valid signature) but is looked up in the `peers: HashMap<PeerId, PeerInfo>` table under a brand-new key that has never received a negative report, so `is_banned()` returns `false` and the connection is treated as a fresh, neutral-reputation peer.
{% endstep %}
{% endstepper %}

## Impact Details

The litep2p peer-reputation/ban system (`client/network/src/litep2p/peerstore.rs`) exists specifically to let a node stop trusting or connecting to a peer that has demonstrated bad behavior. This bug means that protection is not actually keyed on “the peer” in the cryptographic-identity sense the codebase assumes — it is keyed on an attacker-choosable encoding of that identity, so a banned attacker can present the identical Dilithium key again, re-encoded, and be treated as a brand-new, neutral-reputation peer.

I explored, and could not substantiate, several ways this might be escalated beyond a correctness defect into a concretely quantifiable resource-consumption or availability impact.

The central obstacle: this codebase's Dilithium keygen is itself already cheap enough (\~134 microseconds measured) that an attacker who wants a PeerId the ban table has never seen already has that capability with or without this bug, simply by discarding the old key and generating a new one. Every escalation attempt I tried turned out to be achievable at the same cost via ordinary fresh keygen, i.e. not specific to this bug, and/or collapses into the program's explicit Sybil-attack exclusion once multiple concurrent identities become the mechanism rather than serial reconnection.

But PeerId fails to be a deterministic function of the peer's public key, which is a documented, load-bearing property of the scheme. The doc-comment on PeerId itself describes it as “protobuf encoded public key of the peer as specified in \[specs/peer-ids]”, with a plausible but unquantified secondary effect of slightly cheapening ban evasion from “generate a new key, \~134µs” to “re-encode the same key, \~0 additional crypto work”.

## References

* `client/litep2p/src/peer_id.rs` — `PeerId::from_public_key_protobuf`
* `client/litep2p/src/crypto/mod.rs` — `RemotePublicKey::from_protobuf_encoding`
* `client/litep2p/src/crypto/noise/mod.rs` — `parse_and_verify_peer_id`, `handshake()`
* `client/litep2p/src/schema/keys.proto`, `client/litep2p/src/schema/noise.proto`
* `client/network/src/litep2p/peerstore.rs` — `Peerstore`/`PeerstoreHandle`: `is_banned`, `report_peer`
* `client/network/src/litep2p/shim/notification/peerset.rs` — `Peerset`: `max_in`/`max_out` slots keyed on PeerId (checked as an escalation path, ruled out — see Risk Breakdown)
* `client/litep2p/src/transport/manager/peer_state.rs` — `AlreadyConnected` check, also PeerId-keyed (checked as an escalation path, ruled out)
* Protobuf field encoding rules (last-one-wins for singular fields; no wire-order requirement): <https://protobuf.dev/programming-guides/encoding/>
* Prior audit cross-check: grepped `quantus-substrate-audit-20260513` (Equilibrium, 13 May 2026) for `litep2p`, `PeerId`, `noise`, `protobuf` — no matches. This path does not appear to have been covered by the prior audit, which otherwise densely annotates this codebase's other pallets with fixed finding IDs.

## Link to Proof of Concept

<https://gist.github.com/FaLLenSKiLL1/636c2a426b1734eca17be00a86a97575>

## Proof of Concept

Every PoC below runs against the production code of commit `3b243b8`.

{% stepper %}
{% step %}

## PoC 1 (primary): end-to-end ban evasion through the real Peerstore

`https://gist.github.com/FaLLenSKiLL1/636c2a426b1734eca17be00a86a97575`

Environment:

`cargo test -p sc-network`

Requires protoc on PATH for litep2p's build script.

1. Generate a real Dilithium keypair (`litep2p::crypto::dilithium::Keypair::generate()`).
2. Derive its canonical PeerId the same way the real Noise handshake code does.
3. Derive a second, malleated PeerId for the same key (redundant leading Type field).
4. Construct a real Peerstore and call `PeerstoreHandle::report_peer` with `ReputationChange { value: i32::MIN, .. }` against the canonical PeerId — exactly what happens when a connected peer misbehaves badly enough to be banned.
5. Confirm `is_banned(&canonical_peer_id) == true`.
6. Confirm `is_banned(&malleated_peer_id) == false` for the same underlying key — the banned peer, reconnecting with a re-encoded identity payload, is invisible to the ban table.

Actual run output:

`cargo test -p sc-network --lib banned_peer_evades_ban -- --nocapture`

```
running 1 test
test litep2p::peerstore::tests::banned_peer_evades_ban_via_malleated_peer_id_reencoding ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 69 filtered out; finished in 0.00s
```

{% endstep %}

{% step %}

## PoC 2: isolated malleability primitive (library-internal)

`https://gist.github.com/FaLLenSKiLL1/1767e89a0377d70c3eb30019b9ff982b`

Environment:

`cargo test -p litep2p`

Isolates the underlying primitive independent of the Peerstore consumer.

1. Generate a real Dilithium keypair via `Keypair::generate()`.
2. Canonically encode its public key via `PublicKey::to_protobuf_encoding()`.
3. Hand-construct a second, wire-valid encoding of the same message (one redundant leading Type field — protobuf “last field wins”).
4. Decode both byte strings with `RemotePublicKey::from_protobuf_encoding` and confirm they decode to the identical PublicKey.
5. Confirm a signature verifies against the key decoded from either byte string.
6. Compute PeerId for both byte strings and show they differ.

Actual run output:

`cargo test -p litep2p --lib peer_id_malleability_poc -- --nocapture`

```
running 1 test
test crypto::peer_id_malleability_poc::duplicated_type_field_changes_peer_id_but_not_decoded_key_or_signature ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 326 filtered out; finished in 0.01s
```

{% endstep %}

{% step %}

## PoC 3: keygen timing (supports the counterfactual point in Risk Breakdown)

`https://gist.github.com/FaLLenSKiLL1/a673901bf32bc546c1edfd42d9d810b0`

1000 real `Keypair::generate()` calls:

`cargo test -p litep2p --release --lib keygen_timing_poc -- --nocapture`

```
running 1 test
first pubkey len=2592 first_byte=65
Generated 1000 Dilithium ML-DSA-87 keypairs in 131.903217ms (131.903µs/keypair), total_bytes_hashed=2592000
test crypto::dilithium::keygen_timing_poc::time_1000_keygens ... ok

test result: ok. 1 passed; 0 failed; 0 measured; 302 filtered out; finished in 0.13s
```

{% endstep %}
{% endstepper %}
