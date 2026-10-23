# 88305 w a low quantus sdk predicts a multisig address the chain never creates because its derivation omits the pallet s signer deduplication

**Submitted on Aug 12th 2026 at 17:27:44 UTC by @ret2basic for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88305
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions

## Description

## Brief/Intro

`pallet-multisig` derives a multisig account from the **sorted and deduplicated** signer list and documents that determinism as the reason callers may pre-compute the address before creating the multisig. The Quantus SDK is that pre-computation, and it only sorts: neither `quantus_sdk/rust/src/api/multisig.rs` nor the Dart guard in front of it removes or rejects a repeated signer. For any signer list containing a duplicate the SDK returns one address while `create_multisig` succeeds at a different one, and the wallet persists and displays the SDK's answer as the multisig's identity. Funds sent to that displayed address land on a Blake2-256 derived account that has no key and no multisig record, and are permanently unrecoverable.

## Vulnerability Details

### The chain deduplicates, and says so

`pallets/multisig/src/lib.rs:1227-1232` and `pallets/multisig/src/lib.rs:1234-1243`:

```rust
fn normalize_signers(signers: &[T::AccountId]) -> Vec<T::AccountId> {
	let mut sorted = signers.to_vec();
	sorted.sort();
	sorted.dedup();
	sorted
}

/// Derive a deterministic multisig address from signers, threshold, and nonce
///
/// The address is computed as: hash(pallet_id || normalized_signers || threshold || nonce)
/// Signers are automatically sorted and deduplicated internally for deterministic results.
/// This allows users to pre-compute the address before creating the multisig.
pub fn derive_multisig_address(
	signers: &[T::AccountId],
	threshold: u32,
	nonce: u64,
) -> T::AccountId {
	let normalized = Self::normalize_signers(signers);
	Self::derive_multisig_address_inner(&normalized, threshold, nonce)
}
```

Accepting a duplicate-bearing list is deliberate on the chain side; the pallet's own test `create_multisig_deduplicates_signers` (`pallets/multisig/src/tests.rs:249-273`) submits `vec![bob(), bob(), charlie()]` and comments "Should succeed - duplicates are silently removed". `create_multisig` bounds the **raw** length against `MaxSigners`, normalizes, and then checks `normalized_signers.len() >= 2` and `threshold <= normalized_signers.len()`, so `[A, A, B]` with threshold 2 is a valid, successful call.

### The SDK's pre-computation only sorts

`quantus_sdk/rust/src/api/multisig.rs:8-25`:

```rust
fn derive_multisig_address(
    signers: Vec<AccountId32>,
    threshold: u32,
    nonce: u64,
) -> AccountId32 {
    let mut sorted_signers = signers;
    sorted_signers.sort();

    let mut data = Vec::new();
    data.extend_from_slice(&PALLET_ID);
    data.extend_from_slice(&sorted_signers.encode());
    data.extend_from_slice(&threshold.encode());
    data.extend_from_slice(&nonce.encode());

    let hash = BlakeTwo256::hash(&data);
    AccountId32::decode(&mut TrailingZeroInput::new(hash.as_ref()))
        .expect("TrailingZeroInput provides sufficient bytes; qed")
}
```

There is no `dedup()`. Because `Vec<AccountId32>::encode()` length-prefixes and then emits every element, `[A, A, B]` and `[A, B]` produce different preimages and therefore different addresses.

### How the two implementations came apart

The SDK's derivation was correct once. Before commit `1d299f1` ("Address Eiger Multisig Audit - 2", 2026-04-30) the chain **rejected** a duplicate-bearing signer list outright — `create_multisig` sorted, then compared adjacent entries and raised `Error::DuplicateSigner` — while `derive_multisig_address` only sorted. A sort-only client matched that contract exactly, and the May review's `EQ-QNT-MULTISIG-O-02` records precisely those two shapes side by side, observing that "signer normalisation and duplicate checks are split across multiple layers" and recommending they be pushed "into a single authoritative boundary".

That single commit adopted the recommendation on the chain side and, in doing so, changed the contract:

* `DuplicateSigner` and its adjacent-pair check were deleted;
* `normalize_signers` was introduced as "the single authoritative place", sorting **and** deduplicating;
* the doc comment on `derive_multisig_address` was rewritten to "Signers are automatically sorted **and deduplicated** internally for deterministic results. This allows users to pre-compute the address before creating the multisig."

A rejection became a silent normalisation, so the input that used to be refused now succeeds and lands at a different address. The client that performs the pre-computation that sentence promises was not updated with it, and is still the pre-remediation shape. This report is therefore not a restatement of `EQ-QNT-MULTISIG-O-02` — that observation was informational and about code organisation inside one pallet — but a downstream consequence of its remediation, in a different in-scope asset.

### Nothing upstream of it removes the duplicate

`quantus_sdk/lib/src/services/multisig_service.dart:216-223` guards both the prediction (`multisig_service.dart:74`) and the create-call (`multisig_service.dart:117`) with the same predicate, which checks only cardinality:

```dart
void _validateSignersAndThreshold(List<String> signers, int threshold, {required int minSigners}) {
  if (signers.length < minSigners) {
    throw ArgumentError.value(signers, 'signers', 'At least $minSigners signer(s) are required');
  }
  if (threshold < 1 || threshold > signers.length) {
    throw ArgumentError.value(threshold, 'threshold', 'Must be between 1 and ${signers.length}');
  }
}
```

`predictMultisigAddress` forwards the caller's list unchanged to the Rust derivation, and `buildCreateMultisigCall` forwards the same unchanged list into the `create_multisig` extrinsic. So one call site produces the address and the other produces the transaction, from the same unnormalized input, and only the chain normalizes.

The SDK's own two tests for this function — `predict_multisig_address_is_order_independent` and `predict_multisig_address_golden_vector` — both use distinct signers, so no test in the repository ever exercises the duplicate case.

### The asset's own collision-avoidance degrades to a no-op

The divergence is not confined to one function returning a wrong string. `quantus_sdk` ships a routine whose documented job is to avoid address collisions, and on exactly the inputs that trigger this bug it cannot do it.

`quantus_sdk/lib/src/services/multisig_service.dart:83-112` — "Finds the lowest `nonce` whose predicted address is not taken":

```dart
final taken = isAddressTaken ?? isMultisigIndexed;
var nonce = startNonce ?? defaultMultisigNonce;

for (var attempt = 0; attempt < maxAttempts; attempt++) {
  final address = await predict(signers: signers, threshold: threshold, nonce: nonce);
  if (!reservedAddresses.contains(address) && !await taken(address)) {
    return MultisigCreationParams(nonce: nonce, address: address);
  }
  nonce += BigInt.one;
}
```

Every address this loop tests is produced by the same sort-only derivation. For a duplicate-bearing signer list the chain will never create an account at any of them, for any nonce. So `taken(address)` is false on the first iteration and stays false forever: the loop always returns immediately, always declares the default nonce free, and never tests the takenness of the address the chain is actually going to use. The routine is structurally inoperative for precisely the inputs that need it, and it hands the caller the unreachable address back as `MultisigCreationParams.address` — the value callers then persist and display as the multisig's identity.

The same inversion appears in the shipped consumer: `mobile-app/lib/services/multisig_submission_service.dart:120` throws `MultisigAlreadyExistsException` when `isMultisigIndexed(predictedAddress)` is true, so its pre-existence check is also asked about the address that cannot exist rather than the one that will. Neither the SDK's guard nor the app's guard is looking at the account the extrinsic creates.

### Observed divergence

Running the attached PoC at the frozen commits, with `threshold = 2`, `nonce = 0`, and the two signers already used by the SDK's own golden-vector test:

```
sdk_duplicate_address:              qzqAxWFXueDLsJADpbsevSLARiUABFAAKScbhHytLFeGtABDw
chain_duplicate_address:            qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ
sdk_deduplicated_address:           qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ
sdk_no_duplicate_negative_control:  qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ
chain_no_duplicate_negative_control: qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ
```

The chain's address for the duplicate-bearing input equals the SDK's address for the deduplicated input, and equals the value already pinned in the SDK's golden-vector test — so the divergence is attributable to the missing `dedup()` and to nothing else in the encoding, hasher, `PalletId` or SS58 prefix. The negative control confirms the two implementations agree whenever the input has no duplicate.

### How the wrong address becomes the wallet's stored identity

`mobile-app/lib/services/multisig_submission_service.dart:114-126` builds the local account record from the predicted address before submitting:

```dart
final predictedAddress = await service.predictMultisigAddress(
  signers: signers,
  threshold: threshold,
  nonce: effectiveNonce,
);
...
final draft = MultisigAccount(
  name: name,
  accountId: predictedAddress,
  ...
);
```

`MultisigAccount.accountId` is the identity the wallet stores, displays and polls. With a duplicate in the list the extrinsic itself still succeeds, so the submission reports nothing wrong; the wallet then polls the predicted address, never finds the multisig there, and after `MultisigCreationPollingService`'s timeout (`mobile-app/lib/services/multisig_creation_polling_service.dart:49-56`) drops the pending record and shows a generic creation-timeout error. The multisig it just paid to create exists on chain at a different address.

## Why this row, and what it does not claim

The impact list available to this asset has no row for funds lost through a client-side defect with no adversary. I read all four tiers before choosing: arbitrary system commands, retrieving server files, taking down the site, subdomain takeover, direct theft, XSS and HTML injection, acting on behalf of other users, PII disclosure, static content injection, open redirect, expired-link takeover, locking a user out of the site. None of them is written for this.

I selected "Malicious interactions with an already-connected wallet" because one of its enumerated effects is literally what happens here — **substituting the address the connected wallet uses**. `MultisigAccount.accountId` is set to the SDK's predicted address (`mobile-app/lib/services/multisig_submission_service.dart:114`), and the wallet then stores, displays and polls that address as the multisig's identity, while the account the extrinsic actually creates is a different one. The address the connected wallet operates on has been substituted for one that does not exist.

**There is no attacker, and I am not claiming one.** Nothing in this report describes an adversary, a beneficiary, or a malicious transaction. The substitution is produced by the SDK's own derivation, and the funds are not taken by anyone — they are stranded on an account that has no key, no multisig record, and no path by which one could ever be created. I chose the row whose *effect* matches and I am stating plainly that its adjective does not, rather than letting the wording imply an adversary that is not there. "Direct theft of user funds" would have been the alternative; I did not select it because no theft occurs and no one gains.

If the panel judges that the absence of an adversary puts this outside the row it otherwise matches, I ask, as a fallback only, that this be designated an Insight under the programme's Insight Reporting rule rather than closed. I raise the fallback because the programme provides one and I would rather route this correctly than win a categorisation argument; I cannot self-select it, since the rule reserves that designation to Immunefi or the project team.

## Impact Details

The realised harm is that funds sent to the substituted address are unrecoverable, and it is the ceiling rather than the certainty: it is reached exactly when a caller funds the address the SDK returned. Fixing the SDK prevents every future occurrence and needs no hardfork; only recovering funds already sent to a divergent address would require a runtime upgrade.

That address is unrecoverable when it happens. It is `AccountId32::decode(TrailingZeroInput(BlakeTwo256(pallet_id || [A, A, B].encode() || threshold || nonce)))`: not the image of any public key, so no signature can ever authorise a transfer out of it, and with no entry in `Multisig::Multisigs`, so no `propose`/`approve`/`execute` path can move funds from it either. No runtime call recovers an account in this state; only a runtime upgrade could. Nor can that entry appear later: every production derivation reaches `derive_multisig_address_inner` with an already-normalized list — `create_multisig` passes `normalized_signers` (`pallets/multisig/src/lib.rs:457`) and the public `derive_multisig_address` normalizes first (`pallets/multisig/src/lib.rs:1244-1245`) — and `normalize_signers` deduplicates, so no list the chain ever hashes contains a repeat. The preimage `[A, A, B].encode()` is therefore not reachable by any call, which is what makes the freeze permanent rather than merely current: neither the participants nor an attacker can later create a multisig at that address and unlock it. The amount at risk is whatever the participants fund their multisig with, and every later deposit to the same address is lost the same way.

Within the asset under report there is no mitigation at all. `quantus_sdk` accepts the duplicate-bearing list at `predictMultisigAddress`, accepts it again at `buildCreateMultisigCall`, validates only cardinality in the predicate shared by both (`quantus_sdk/lib/src/services/multisig_service.dart:216-223`), and — as shown above — runs its collision-avoidance loop against addresses the chain will never create. The SDK is a library published for integrators; its public API is its attack surface, and nothing in it rejects or normalises the input.

The one guard that exists lives in a **different in-scope asset** (`.../mobile-app`, listed separately on the scope page) and is a string comparison in a single view, not a property of the SDK. I record it because it is true and it limits what one particular consumer will do, not because it bounds the defect in the asset I am reporting:

* `mobile-app/lib/v2/screens/multisig/add_multisig_screen.dart:119` rejects a repeated signer in `_addSigner` before any prediction is made, so the wallet's own creation screen cannot produce the divergent input.
* Even if it could, the wallet never presents the predicted address as a funding target. `MultisigCreationPollingService` polls it, times out, and reports a creation-timeout error (`mobile-app/lib/services/multisig_creation_polling_service.dart:55`), and `MultisigService.discoverForUser` finds multisigs by querying the indexer for records where the user is a signer and takes the address from the record's own `id` — the chain's address (`quantus_sdk/lib/src/services/multisig_service.dart:57`). So the real multisig stays discoverable and the funds stay reachable.

The exposure is therefore the SDK as a distributed library rather than the wallet as a shipped app. `MultisigService.predictMultisigAddress` is documented as predicting the on-chain multisig address and is the SDK's only API for doing so; the chain's own `derive_multisig_address` documents deduplication as precisely what makes that pre-computation possible. A third-party integrator calling it as documented, with a signer list the SDK's validator and the chain's extrinsic both accept, receives an address the chain will never create, with no error and no signal that anything is wrong — and the failure only becomes visible after value has been sent somewhere unrecoverable.

## References

* <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* <https://github.com/immunefi-team/audit-comp-quantus-chain>
* <https://docs.rs/parity-scale-codec/latest/parity\\_scale\\_codec/trait.Encode.html>
* <https://docs.rs/sp-runtime/latest/sp\\_runtime/traits/struct.TrailingZeroInput.html>

## Proof of Concept

### Prerequisites

* A Rust toolchain able to build the frozen SDK crate's dependencies (the PoC pins `rust-toolchain` from the `quantus_sdk/rust` workspace; stable 1.93 was used).
* A local checkout of the in-scope SDK asset at the frozen competition commit:
  * `https://github.com/immunefi-team/audit-comp-quantus-apps` branch `audit-comp-ready`, commit `d5fc4d6572ebc2244e95009a33f5da5048935867`.
* The chain-side reference for the comparison is `https://github.com/immunefi-team/audit-comp-quantus-chain` branch `audit-comp-ready`, commit `3b243b870a5442a6e5f443056f67bbba87d472f5`, file `pallets/multisig/src/lib.rs`. That repository is a different asset and is not checked out here, so the harness reimplements the pallet's derivation locally from that source and pins the `PalletId` bytes it uses; the negative control below is what proves the reimplementation faithful.
* No network access, no chain node, no funded account and no key material is required. Nothing outside the PoC directory is modified: the harness includes the frozen SDK file `quantus_sdk/rust/src/api/multisig.rs` verbatim with `#[path]` and only stubs the `#[flutter_rust_bridge::frb(sync)]` attribute so the file compiles without the Flutter bridge.
* The optional Dart reachability check additionally needs the Flutter SDK and `quantus_sdk`'s own `pubspec.lock` dependencies. The Rust harness alone is sufficient to establish the finding.

### Reproduction steps

{% stepper %}
{% step %}

## Place the PoC files

Place the PoC files at the paths shown below, relative to the root of the `audit-comp-quantus-apps` checkout.
{% endstep %}

{% step %}

## Run the harness

From that root, run the harness:

```bash
cargo run --manifest-path poc/multisig_duplicate_derivation/Cargo.toml --locked --offline
```

{% endstep %}

{% step %}

## Observe the output

Observe the printed addresses and the final `assertions: PASS` line. The harness asserts, and the run reproduces:

```
sdk_duplicate_address:               qzqAxWFXueDLsJADpbsevSLARiUABFAAKScbhHytLFeGtABDw
chain_duplicate_address:             qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ
sdk_deduplicated_address:            qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ
sdk_no_duplicate_negative_control:   qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ
chain_no_duplicate_negative_control: qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ
assertions: PASS
```

The three relations that matter are all asserted by the harness, not merely printed:

* `SDK([A, A, B], 2, 0) != CHAIN([A, A, B], 2, 0)` — the SDK predicts an address the chain does not create.
* `CHAIN([A, A, B], 2, 0) == SDK([A, B], 2, 0)` — the chain's answer is exactly the SDK's answer for the deduplicated list, so the missing `dedup()` is the whole of the difference.
* `SDK([B, A], 2, 0) == CHAIN([B, A], 2, 0)` — negative control: with no duplicate in the input the two implementations agree, which rules out an unrelated mismatch in the SCALE encoding, the hasher, the `PalletId` or the SS58 prefix.

`qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ` is also the value pinned by the SDK's own `predict_multisig_address_golden_vector` test, so the chain-side reimplementation agrees with the SDK on every input the SDK already has a fixture for.
{% endstep %}

{% step %}

## Optionally confirm Dart reachability

Optionally, confirm that the SDK's public Dart surface accepts the duplicate-bearing list rather than rejecting it, from the same checkout root:

```bash
flutter test quantus_sdk/test/poc_multisig_duplicate_reachability_test.dart
```

The first case asserts that `MultisigService.buildCreateMultisigCall` accepts `[A, A, B]` and emits the extrinsic with all three entries preserved — that is the call the chain then normalizes to `[A, B]`. The second asserts that `MultisigService.predictMultisigAddress` raises no `ArgumentError` for the same list, i.e. the shared `_validateSignersAndThreshold` predicate does not reject duplicates for either `minSigners` value used in the service.
{% endstep %}
{% endstepper %}

### Complete runnable PoC files

Each code block below is the complete file named in its heading. Save the blocks under `poc-inputs/` using those paths, then follow the reproduction steps above.

#### `poc-inputs/poc/multisig_duplicate_derivation/Cargo.toml`

```toml
[package]
name = "quantus_multisig_duplicate_derivation_poc"
version = "0.1.0"
edition = "2021"
publish = false

[dependencies]
flutter_rust_bridge = { path = "frb_attr_stub" }
parity-scale-codec = { version = "3.7.5", default-features = false, features = ["std"] }
sp-core = { path = "sp_core_stub" }
sp-runtime = { path = "sp_runtime_stub" }
```

#### `poc-inputs/poc/multisig_duplicate_derivation/frb_attr_stub/Cargo.toml`

```toml
[package]
name = "flutter_rust_bridge"
version = "2.12.0"
edition = "2021"
publish = false

[lib]
proc-macro = true
```

#### `poc-inputs/poc/multisig_duplicate_derivation/frb_attr_stub/src/lib.rs`

```rust
extern crate proc_macro;

use proc_macro::TokenStream;

#[proc_macro_attribute]
pub fn frb(_attr: TokenStream, item: TokenStream) -> TokenStream {
    item
}
```

#### `poc-inputs/quantus_sdk/test/poc_multisig_duplicate_reachability_test.dart`

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/generated/planck/types/pallet_multisig/pallet/call.dart' show CreateMultisig;
import 'package:quantus_sdk/generated/planck/types/quantus_runtime/runtime_call.dart';
import 'package:quantus_sdk/src/services/multisig_service.dart';
import 'package:quantus_sdk/src/services/substrate_service.dart' show getAccountId32;

void main() {
  group('PoC: duplicate signer reachability', () {
    const signerA = '5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY';
    const signerB = '5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty';

    test('public create-call path accepts and preserves duplicate signer list', () {
      final call = MultisigService().buildCreateMultisigCall(
        signers: const [signerA, signerA, signerB],
        threshold: 2,
        nonce: BigInt.zero,
      );

      final create = call.value0 as CreateMultisig;
      expect(create.threshold, 2);
      expect(create.nonce, BigInt.zero);
      expect(create.signers, [getAccountId32(signerA), getAccountId32(signerA), getAccountId32(signerB)]);
    });

    test('public prediction path does not reject duplicate signer list during validation', () async {
      const duplicateSigners = [signerA, signerA, signerB];
      Object? error;

      try {
        await MultisigService().predictMultisigAddress(signers: duplicateSigners, threshold: 2);
      } catch (e) {
        error = e;
      }

      expect(error, isNot(isA<ArgumentError>()));
    });
  });
}
```

#### `poc-inputs/poc/multisig_duplicate_derivation/src/main.rs`

```rust
use parity_scale_codec::{Decode, Encode};
use sp_core::crypto::{AccountId32, Ss58AddressFormat, Ss58Codec};
use sp_runtime::traits::{BlakeTwo256, Hash as HashT, TrailingZeroInput};

#[path = "../../../quantus_sdk/rust/src/api/multisig.rs"]
#[rustfmt::skip]
mod frozen_sdk_multisig;

const PALLET_ID: [u8; 8] = *b"py/mltsg";
const SS58_PREFIX: u16 = 189;
const THRESHOLD: u32 = 2;
const NONCE: u64 = 0;

const SIGNER_A: &str = "5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY";
const SIGNER_B: &str = "5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty";

fn chain_normalize_signers(mut signers: Vec<AccountId32>) -> Vec<AccountId32> {
    signers.sort();
    signers.dedup();
    signers
}

fn chain_derive_multisig_address(
    signers: Vec<AccountId32>,
    threshold: u32,
    nonce: u64,
) -> AccountId32 {
    let normalized_signers = chain_normalize_signers(signers);

    let mut preimage = Vec::new();
    preimage.extend_from_slice(&PALLET_ID);
    preimage.extend_from_slice(&normalized_signers.encode());
    preimage.extend_from_slice(&threshold.encode());
    preimage.extend_from_slice(&nonce.encode());

    let hash = BlakeTwo256::hash(&preimage);
    AccountId32::decode(&mut TrailingZeroInput::new(hash.as_ref()))
        .expect("TrailingZeroInput provides enough bytes for AccountId32")
}

fn decode_signers(signers: &[&str]) -> Vec<AccountId32> {
    signers
        .iter()
        .map(|signer| {
            AccountId32::from_ss58check(signer).expect("fixture signers are valid SS58 addresses")
        })
        .collect()
}

fn sdk_predict(signers: &[&str]) -> String {
    frozen_sdk_multisig::predict_multisig_address(
        signers.iter().map(|signer| signer.to_string()).collect(),
        THRESHOLD,
        NONCE,
    )
    .expect("SDK prediction should accept fixture signers")
}

fn chain_predict(signers: &[&str]) -> String {
    chain_derive_multisig_address(decode_signers(signers), THRESHOLD, NONCE)
        .to_ss58check_with_version(Ss58AddressFormat::custom(SS58_PREFIX))
}

fn main() {
    let duplicate_signers = [SIGNER_A, SIGNER_A, SIGNER_B];
    let deduplicated_signers = [SIGNER_A, SIGNER_B];
    let no_duplicate_signers = [SIGNER_B, SIGNER_A];

    let sdk_duplicate = sdk_predict(&duplicate_signers);
    let sdk_deduplicated = sdk_predict(&deduplicated_signers);
    let chain_duplicate = chain_predict(&duplicate_signers);
    let chain_no_duplicate = chain_predict(&no_duplicate_signers);
    let sdk_no_duplicate = sdk_predict(&no_duplicate_signers);

    println!("Quantus multisig duplicate derivation PoC");
    println!("asset: quantus_sdk");
    println!("target_head: d5fc4d6572ebc2244e95009a33f5da5048935867");
    println!("chain_reference_release: audit-comp-quantus-chain@3b243b870a5442a6e5f443056f67bbba87d472f5");
    println!("unavoidable_deviation: chain pallet source is not present in this apps worktree, so the chain-side derivation is reimplemented locally. The reimplementation was afterwards checked line by line against the chain pallet source at pallets/multisig/src/lib.rs:1249-1272 (audit-comp-quantus-chain@3b243b870a5442a6e5f443056f67bbba87d472f5): same PalletId bytes, same preimage order (pallet_id || signers.encode() || threshold.encode() || nonce.encode()), same BlakeTwo256 hashing, same TrailingZeroInput decode, and normalize_signers is sort+dedup as reimplemented here.");
    println!(
        "pallet_id_ascii: {}",
        std::str::from_utf8(&PALLET_ID).expect("ASCII pallet id")
    );
    println!("pallet_id_bytes: {:?}", PALLET_ID);
    println!("ss58_prefix: {SS58_PREFIX}");
    println!("threshold: {THRESHOLD}");
    println!("nonce: {NONCE}");
    println!("duplicate_input: [A, A, B]");
    println!("signer_a: {SIGNER_A}");
    println!("signer_b: {SIGNER_B}");
    println!("sdk_duplicate_address: {sdk_duplicate}");
    println!("sdk_deduplicated_address: {sdk_deduplicated}");
    println!("chain_duplicate_address: {chain_duplicate}");
    println!("sdk_no_duplicate_negative_control: {sdk_no_duplicate}");
    println!("chain_no_duplicate_negative_control: {chain_no_duplicate}");

    assert_ne!(
        sdk_duplicate, sdk_deduplicated,
        "SDK must hash the duplicate-bearing signer vector differently from the deduplicated vector"
    );
    assert_ne!(
        sdk_duplicate, chain_duplicate,
        "SDK duplicate-bearing prediction must diverge from chain-normalized derivation"
    );
    assert_eq!(
        chain_duplicate, sdk_deduplicated,
        "chain duplicate-bearing derivation must equal the SDK result for the explicitly deduplicated signer list"
    );
    assert_eq!(
        sdk_no_duplicate, chain_no_duplicate,
        "negative control: SDK and chain agree when the signer list contains no duplicates"
    );

    println!("assertions: PASS");
}
```
