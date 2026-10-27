# 89999 w a critical multisig discovery accepts an indexer supplied address without verifying its derivation

**Submitted on Aug 22nd 2026 at 12:00:40 UTC by @bytes for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89999
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions

## Description

### Brief/Intro

A Quantus multisig's on-chain address is fully determined by its signers, threshold, and nonce, and the wallet ships that derivation and runs it whenever it creates a multisig. When the wallet instead discovers a multisig that already exists, it accepts the address the indexer reports without recomputing it, even though the discovery record already carries the signers, threshold, and nonce the derivation needs. A compromised or malicious discovery indexer can return a record that lists the victim among the signers, so it appears to be the user's own multisig, while supplying an attacker-controlled address in the address field. The wallet stores, displays, and transacts against that address as the multisig, so funds the user sends to what they believe is their own shared account are sent to an account the attacker controls.

### Vulnerability Details

The multisig address is computed as `hash(pallet_id, sorted_signers, threshold, nonce)`. The SDK ships this derivation in `quantus_sdk/rust/src/api/multisig.rs`: `predict_multisig_address` (`:31`) implements it, and a golden-vector test (`:76`) pins that signers `[5Grw..., 5FHn...]`, threshold 2, nonce 0 derive to `qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ`. The Dart entry point is `predictMultisigAddress` in `quantus_sdk/lib/src/services/multisig_service.dart:73`.

When the wallet creates a multisig it uses that derivation and never trusts an externally supplied address: `resolveMultisigCreationParams` (`multisig_service.dart:87`) computes the address with `predictMultisigAddress` at `:105`.

Discovery does not. `discoverForUser` (`multisig_service.dart:44`) reads each record's address directly from `record['id']` (`:57`) and builds a local account with `multisigAccountFromIndexerRecord` (`:64`). That mapper takes the address verbatim and stores it as the account's identity, while reading the signers, threshold, and nonce from the same record:

```dart
static MultisigAccount multisigAccountFromIndexerRecord(Map<String, dynamic> record, {required String myMemberAccountId, required String name}) {
  final address = stringFromJson(record['id']);                                    // :169 taken as-is
  final signers = nonEmptyStringListFromJson(record['signers'], 'signers');        // :171
  final threshold = multisigThresholdFromJson(record['threshold'], signerCount: signers.length); // :172
  return MultisigAccount(
    name: name,
    accountId: address,                                                            // :176 stored as the account identity
    signers: signers,
    threshold: threshold,
    nonce: bigIntFromJson(record['nonce']),                                        // :179
    myMemberAccountId: myMemberAccountId,
  );
}
```

Every input `predictMultisigAddress` requires is present in this function, yet it is never called, and there is no point on the discovery path where `record['id']` is compared against the derivation. The wallet authenticates the address it creates and accepts on trust the address it discovers, using the same data and the same primitive it already holds.

The accepted address is then used as the multisig throughout the app. The discovered account is persisted keyed on it (`mobile-app/lib/v2/screens/multisig/discover_multisig_screen.dart:26` calls the accounts provider's `add` at `:35`, which stores it via `SettingsService.addMultisigAccount`, `quantus_sdk/lib/src/services/settings_service.dart:269`). It is displayed to the user as the multisig's address (`discover_multisig_screen.dart:217`). It becomes the account identity that later actions transact against: the approve, execute, and cancel confirmation sheets all key on `msig.accountId` (`multisig_approve_confirm_sheet.dart:29`, `multisig_execute_confirm_sheet.dart:23`, `multisig_cancel_confirm_sheet.dart:25`).

The broad risk of trusting indexer-sourced multisig data is already recognized; the review on PR #509 rates it High. What is specific here is narrower: the multisig address itself is never checked against the derivation the same SDK ships and already uses on the create path, so a hostile indexer controls the one field that determines where value settles while the signer list and threshold it also supplies serve only as reassuring display.

A runnable test that drives the wallet's real discovery flow against a controlled indexer response, and shows discovery returns an address that does not match the derivation for the same inputs, is provided in the proof-of-concept section.

### Impact Details

A user who runs discovery against a compromised or malicious indexer can be shown, and can add, an account presented as their own multisig whose address is attacker-controlled. The signer list and threshold rendered in the UI are attacker-supplied display values; the address is the only field that determines where funds settle, and it belongs to an ordinary account the attacker controls. The multiple-approval protection the interface implies does not constrain that account, because it is not a multisig at all.

Any value routed to the substituted address is lost to the attacker: deposits the user makes to their multisig, and funds a counterparty sends after the user shares the address as their shared account. The loss is bounded only by how much value reaches the substituted address before the substitution is noticed, and nothing in the wallet surfaces the discrepancy, because the wallet held the signers, threshold, and nonce needed to detect it and never checked. This is a malicious interaction with an already-connected wallet in which an attacker-controlled address is substituted for the user's multisig, the in-scope impact selected for this report.

The precondition is control of the discovery indexer or its responses. The endpoint is served over HTTPS, so this is indexer compromise, not passive network interception.

## References

* Derivation and golden vector: `quantus_sdk/rust/src/api/multisig.rs:29`, `:31`, `:76`
* Derivation entry point and create path: `quantus_sdk/lib/src/services/multisig_service.dart:73`, `:87`, `:105`
* Discovery path and the unchecked mapping: `quantus_sdk/lib/src/services/multisig_service.dart:44`, `:57`, `:64`, `:164`, `:169`, `:176`
* Persisted, displayed, and transacted against: `mobile-app/lib/v2/screens/multisig/discover_multisig_screen.dart:26`, `:35`, `:217`; `quantus_sdk/lib/src/services/settings_service.dart:269`; `mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart:29`
* Public context: PR #509 review
* Assets: `quantus_sdk` and `mobile-app`, commit `d5fc4d6`

## Link to Proof of Concept

<https://gist.github.com/therealbytes/7d871d6d38c799f0902fd341a00745fd>

## Proof of Concept

The proof of concept runs the wallet's real discovery flow against a controlled indexer response and shows the flow returns a multisig whose address is the one the indexer supplied, not the address the wallet's own derivation produces from the same signers, threshold, and nonce, and does not reject the record.

### What it does

{% stepper %}
{% step %}

## A controlled indexer response

A loopback HTTP server replies to the discovery query with one record: the signers include the victim, the threshold is 2, the nonce is set, and the address field is an attacker-selected valid address that is not the address these inputs derive to.
{% endstep %}

{% step %}

## Run the wallet discovery flow

The wallet's discovery endpoint is pointed at the loopback server, and the wallet's own `discoverForUser` is called for the victim. All network handling, parsing, and account construction is the wallet's code; only the server response is controlled.
{% endstep %}

{% step %}

## Compare the returned address

The address returned by discovery is compared against the derived address.
{% endstep %}
{% endstepper %}

### What it asserts

* The wallet posted its real discover query to the endpoint (method and query body are checked).
* Discovery returns an account whose address equals the attacker-selected address.
* That address differs from the derived address for the same signers, threshold, and nonce.
* Discovery returns successfully without rejecting the record.

The derived address is not computed by this test. It was established by the wallet's Rust golden-vector test for the same signers, threshold, and nonce, and is used here as the control value.

### How to run

{% stepper %}
{% step %}

## Run the discovery test

Download the test from the gist below[^1] and save it into the SDK package, then run it against the pinned commit:

```bash
mkdir -p quantus_sdk/test/poc
# save the gist file as quantus_sdk/test/poc/multisig_discovery_unverified_test.dart
cd quantus_sdk
flutter test test/poc/multisig_discovery_unverified_test.dart
```

{% endstep %}

{% step %}

## Run the Rust golden-vector test

The control value is established separately by the wallet's own Rust test:

```bash
cd quantus_sdk/rust
cargo test -p rust_lib_quantus_wallet predict_multisig_address_golden_vector -- --nocapture
```

{% endstep %}
{% endstepper %}

### Result

```
00:00 +0: loading test/poc/multisig_discovery_unverified_test.dart
00:00 +0: discovery returns an indexer address that does not match the derivation
[poc] returned account address: qzkx7Uk9bJu5Eayk8LnG99Vt2DukQukoso7L99vs2yPnT5Ny5
[poc] derived control address:  qzkvQ2YBa7RrvYKNCXxJw7hJZPBSi6GzZxXGN2fN6DddaH5HJ  (from the Rust golden-vector test)
[poc] discovery returned the record successfully, without rejecting it
00:00 +1: control: an honest record is returned the same way, so the attack case is not rejected incidentally
00:00 +2: All tests passed!
```

The independently run Rust golden-vector test establishing the control address produced:

```
    Finished `test` profile [unoptimized + debuginfo] target(s) in 2.33s
     Running unittests src/lib.rs (target/debug/deps/rust_lib_quantus_wallet-0c0dc9f411399f7d)

running 1 test
test api::multisig::tests::predict_multisig_address_golden_vector ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 12 filtered out; finished in 0.00s
```

[^1]: Gist: <https://gist.github.com/therealbytes/7d871d6d38c799f0902fd341a00745fd>
