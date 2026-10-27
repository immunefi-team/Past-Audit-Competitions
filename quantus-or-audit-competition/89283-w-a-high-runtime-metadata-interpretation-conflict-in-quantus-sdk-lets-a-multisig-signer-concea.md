# 89283 w a high runtime metadata interpretation conflict in quantus sdk lets a multisig signer conceal transfer all and drain shared funds

**Submitted on Aug 18th 2026 at 12:01:15 UTC by @evolutionstorm for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89283
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Direct theft of user funds

## Description

## Summary

The in-scope Quantus SDK bundles a spec-136 generated runtime decoder while the current chain uses spec 143. A crafted nested Utility call is valid and fully consumed by both schemas, yet the two schemas assign different child boundaries and different effects to the same bytes. The listed Quantus mobile app uses this SDK decoder to render the authoritative bytes of an on-chain multisig proposal immediately before enabling approval. A malicious signer can therefore make a co-signer approve one displayed call tree while the current runtime executes another.

The SDK's generated decoder displays:

```
Utility.force_batch([
  Balances.transfer_allow_death(Index(1), 872415488),
  System.remark(<29 attacker-selected opaque bytes>)
])
```

The current runtime executes:

```
Utility.force_batch([
  Balances.transfer_allow_death(Index(()), 1),        // fails account lookup
  Balances.transfer_all(Id(<attacker account>), false) // executes and drains the sender
])
```

`force_batch` deliberately continues after the first child fails. The second child therefore transfers the shared multisig account's entire spendable balance to an ML-DSA-65 account controlled by the attacker. An independent `Utility.if_else` witness produces the same theft through its fallback branch.

The included current-runtime regression executes the complete 2-of-2 propose, approve, and execute lifecycle and proves attacker spend-out. The included Flutter widget regression uses the real mobile confirmation sheet and real SDK decoder to assert that the authoritative witness bytes render as the benign spec-136 tree, no runtime-version warning appears, approval remains enabled, and the same bytes are handed to submission without canonical re-encoding.

## Primary exploit path: listed mobile-app multisig approval

The listed mobile application uses the vulnerable Quantus SDK decoder for a live multisig authorization display:

{% stepper %}
{% step %}

## `CallDecoder.decodeBytes`

`quantus_sdk/lib/src/chain/call_decoder.dart:48-60` implements `CallDecoder.decodeBytes`. It decodes through the generated `runtime.RuntimeCall.codec` and requires the bytes to fit exactly.
{% endstep %}

{% step %}

## Stale generated runtime

That SDK decoder imports the stale generated Planck runtime and `sp_runtime/multiaddress/multi_address.dart` at `quantus_sdk/lib/src/chain/call_decoder.dart:24-40`.
{% endstep %}

{% step %}

## Recursive multisig call decoding

For multisig `propose` and `approve`, the SDK recursively decodes the stored inner call and labels it as the proposed or approved call at `quantus_sdk/lib/src/chain/call_decoder.dart:313-353`.
{% endstep %}

{% step %}

## Proposal storage

Any existing multisig signer can supply the bounded inner-call bytes to `propose`. The current runtime verifies that they decode and fit the weight/policy limits, then stores those exact bytes in `ProposalData.call` at `pallets/multisig/src/lib.rs:535-623,674-691`. A normal multisig is not restricted by the high-security whitelist.
{% endstep %}

{% step %}

## Mobile app display

The mobile app fetches the proposal's authoritative on-chain call bytes and passes them to `CallDecoder.decodeBytes` at `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:145-195`.
{% endstep %}

{% step %}

## Approval enablement

The confirmation sheet renders the decoded headline and every decoded field, then enables the approval action at `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:345-405`.
{% endstep %}

{% step %}

## Byte-preserving approval submission

The approval flow retrieves those bytes from chain and supplies the same bytes to `buildApproveCall` and submission at `mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart:38-64`.
{% endstep %}

{% step %}

## Runtime execution

The pallet accepts the approval only when its resubmitted bytes equal the stored payload byte-for-byte at `pallets/multisig/src/lib.rs:740-782`. Once the threshold is met, any signer can call `execute`; it decodes the stored payload under the current runtime and dispatches it as the multisig account at `pallets/multisig/src/lib.rs:1068-1156`.
{% endstep %}
{% endstepper %}

The official app's proposal builder currently exposes a transfer-shaped helper, but this does not constrain the public `multisig.propose` extrinsic. A malicious signer can submit the crafted nested call through any compatible client. The second signer then receives the authoritative attacker bytes from chain, sees the stale SDK's benign interpretation, and signs an approval that is bound to those exact bytes. The same bytes survive proposal storage, approval, and execution; only their meaning differs between the SDK display and the current runtime.

The runtime PoC exercises this complete 2-of-2 path. It creates and funds a normal multisig, proposes the witness as the attacker signer, asserts byte-identical storage, approves it as the victim signer, executes it as the attacker, observes the shared account drained, and signs a final spend from the attacker destination. A separate mobile widget regression covers the exact display-and-approval behavior at the in-scope application boundary.

### The multisig path shows no version warning at all

The cold-wallet consumer at least renders a generic runtime-mismatch banner. The listed mobile-app multisig approval flow renders none: `mobile-app/lib/v2/screens/multisig/` contains no reference to `specMatchesBundled`, `specVersion`, `runtimeVersion`, or any mismatch notice. The approving signer is shown a decoded call tree and an approval action, with nothing indicating that the bundled decoder and the live runtime disagree.

### Preconditions for the in-scope multisig path

{% stepper %}
{% step %}

## Attacker access

The attacker is one existing signer of an ordinary multisig. Normal multisigs are not restricted by the high-security whitelist.
{% endstep %}

{% step %}

## Victim approval

The victim co-signer approves the proposal their app displays.
{% endstep %}
{% endstepper %}

No compromised device, no dismissed warning, no user error against a visible caution, and no victim key material are required. The attacker possesses one legitimate signer key but lacks the threshold authority required to approve or execute the proposal alone. The funds at risk are the shared multisig account's entire spendable balance.

### This is not a privileged actor acting within their privileges

The scope page excludes "actions by privileged actors within their privileges." That exclusion does not apply here.

A multisig signer is entitled to propose a call. A multisig signer is not entitled to cause a co-signer to approve a call the co-signer was never shown. The threshold exists precisely to require the second signer's informed consent, and the Quantus multisig pallet documents that intent in its own header at `pallets/multisig/src/lib.rs:1-12`:

> Approve proposed transactions (approvers resubmit the inner call, which must match the stored proposal payload — binding each approval signature to the actual call)

That binding holds byte-for-byte and fails semantically. The approver's signature is bound to bytes whose executed meaning was never displayed to them, so the pallet's stated guarantee is satisfied while the consent it exists to capture is not.

The decisive test is repair: with the SDK decoding under authenticated current-runtime metadata, the same one-of-two signer, on the same multisig, cannot obtain the co-signer's informed approval for the hidden drain. The attack crosses the threshold boundary; it is not a signer exercising threshold authority they already possess.

## Impact

The SDK interpretation conflict lets the listed mobile app present a small indexed transfer plus harmless remark data for bytes that the current runtime executes as `Balances.transfer_all(..., keep_alive=false)` to an attacker-controlled account. The affected balance is the shared multisig account's entire spendable balance, not the approving signer's separate personal account.

The positive PoC starts a victim with `1,000,000,000,000,000` units and proves:

* the signed extrinsic passes ML-DSA-65 signature validation and all transaction extensions;
* the first current-runtime child fails as expected;
* the continuing/fallback wrapper executes the hidden `transfer_all`;
* the attacker receives the victim's spendable balance, less transaction fees; and
* the attacker signs a second extrinsic and spends the received funds onward.

This is direct theft, not a denial of service or transfer to an unspendable address.

The multisig regression independently proves:

* the attacker signer controls the exact inner bytes supplied to `propose`;
* proposal storage preserves all 43 witness bytes;
* the victim's `approve` call is accepted only with that exact stored payload;
* `execute` dispatches those bytes as the shared multisig account;
* the nested `force_batch` witness drains the multisig balance to the attacker; and
* the retained attacker key spends the received balance onward.

## Root cause

### 1. The Quantus SDK and current runtime consume `MultiAddress::Index` differently

The in-scope SDK's generated spec-136 Dart type stores an `AccountIndex` and decodes a compact integer:

```
quantus_sdk/lib/generated/planck/types/sp_runtime/multiaddress/multi_address.dart:152-173
```

The current runtime instead configures:

```rust
type Lookup = sp_runtime::traits::AccountIdLookup<Self::AccountId, ()>;
```

at:

```
runtime/src/configs/mod.rs:110-114
```

Consequently, old `Index(1)` consumes the variant byte and one compact-value byte, while current `Index(())` consumes only the variant byte. In a recursive sequence, every later byte is decoded from a different cursor.

The SDK already models this exact threat and defends against only one of its symptoms. `CallDecoder.decodeBytes` documents its own reasoning at `quantus_sdk/lib/src/chain/call_decoder.dart:48-52`:

```dart
/// Trailing bytes mean the sender and this decoder disagree about the call's
/// shape, so the result cannot be trusted for display — throw instead.
```

The author recognised that a sender and this decoder can disagree about a call's shape, and that such a result must not be displayed. The implemented check is a trailing-byte test. Both witnesses in this report are constructed to consume every byte under both schemas, so the disagreement the comment describes occurs while the guard reports success. The defence is present but incomplete for the case it names.

### 2. Recursive wrappers turn the cursor shift into executable behavior

The outer Utility pallet/call indices and recursive container shapes are valid in both versions. The shifted bytes form a second current-runtime call rather than trailing garbage.

`force_batch` records `ItemFailed` and continues over later children:

```
pallets/utility/src/lib.rs:461-508
```

`if_else` dispatches its fallback after the main call fails:

```
pallets/utility/src/lib.rs:568-635
```

The first current child fails because `AccountIdLookup<AccountId, ()>` cannot resolve `MultiAddress::Index(())`. That failure becomes the branch primitive that reaches the reframed `transfer_all` child.

## Exact witnesses

### `force_batch` drain

```
09040802000104020400d0000074b3aed30cdbb7c86d2da943ee944a3ee82ef086e6dfddc861f5df771600
```

Length: 43 bytes. Both call decoders consume all 43 bytes and round-trip the input exactly.

### `if_else` drain

```
090602000104020400d0000074b3aed30cdbb7c86d2da943ee944a3ee82ef086e6dfddc861f5df771600
```

Length: 42 bytes. Both call decoders consume all 42 bytes and round-trip the input exactly.

### Attacker-controlled destination

```
account=d0000074b3aed30cdbb7c86d2da943ee944a3ee82ef086e6dfddc861f5df7716
seed=fb760c00000000000000f6ed18000000975635f409d131678e010000000060df
```

The seed is synthetic PoC material. It is retained so the runtime test can prove the destination is spendable by the attacker.

## Recommended remediation

{% stepper %}
{% step %}

## Authenticate decoder metadata

Authenticate and bind the SDK decoder metadata to the connected chain's genesis hash, spec version, and transaction version before using decoded data for authorization.
{% endstep %}

{% step %}

## Block mismatched multisig approval and execution

In the mobile multisig flow, block approval and execution whenever the authenticated live runtime does not match the decoder metadata. Do not fall back to an indexer display or a warning.
{% endstep %}

{% step %}

## Regenerate SDK bindings

Regenerate and ship the Quantus SDK Planck bindings for current spec 143 metadata, including `MultiAddress<AccountId, ()>`.
{% endstep %}

{% step %}

## Canonically re-encode reviewed calls

Canonically re-encode the reviewed call under the authenticated current metadata and require a byte-for-byte match with the proposal bytes before enabling approval.
{% endstep %}

{% step %}

## Add regression coverage

Add both nested witnesses to SDK decoder tests and keep the included mobile widget regression so a mismatch cannot render a benign tree, remain warning-free, and reach the submit callback.
{% endstep %}
{% endstepper %}

## Public prior art, incomplete fix, and spec-143 recurrence

Quantus Apps PR 588 and commit `b491e8771943eb3ed795bc0f11d56b03327285c0` are directly relevant public prior art. They disclose that the generated Planck metadata had drifted from the live chain, identify the bundled runtime as spec 136 / transaction version 3, and explain that stale metadata can make the same call index decode to a different call or display incorrect labels and parameters.

The same change also establishes the multisig authorization flow used here. It adds the `call: Vec<u8>` field to `multisig.approve`; documents that an approver must resubmit the proposal's inner call so the signature is bound to the actual approved bytes; introduces the shared SDK `CallDecoder` built on generated Polkadart codecs; and changes approval to load the byte-compared call from authoritative chain storage and block confirmation until that load succeeds.

This report should therefore be assessed as an exploitable recurrence after an incomplete fix, not as discovery of the general metadata-drift class or of the approve/decode/inspect/byte-bind flow. PR 588 repaired the then-observed drift by regenerating static bindings to spec 136. It did not bind the decoder to authenticated live metadata, compare the live spec or metadata hash before rendering, or gate mobile multisig approval on runtime-version equality. The chain subsequently advanced to spec 143 while the SDK remained at spec 136, recreating the same trust-boundary failure in the flow PR 588 introduced.

The additional security results demonstrated here are narrower and concrete:

{% stepper %}
{% step %}

## Current spec-143 recurrence

The recurrence exists at current spec 143 in the submitted revisions.
{% endstep %}

{% step %}

## `MultiAddress::Index` conflict

The conflict is inside the SDK's generated `MultiAddress::Index` argument: spec 136 consumes a compact integer where the current runtime consumes unit `()`.
{% endstep %}

{% step %}

## Recursive Utility call reframing

That one-byte child-boundary shift occurs inside recursive Utility calls and turns bytes displayed as inert `System.remark` data into an additional executable `Balances.transfer_all` child.
{% endstep %}

{% step %}

## Exact-fit bypass

Both schemas consume and round-trip every byte. The shared decoder's explicit exact-fit/trailing-byte guard therefore reports success instead of rejecting the disagreement.
{% endstep %}

{% step %}

## Demonstrated theft

The current spec-143 runtime accepts the byte-bound proposal and approval, dispatches the hidden call as the shared account, drains its balance, and leaves the proceeds spendable by the attacker.
{% endstep %}
{% endstepper %}

Neither PR 588 nor its commit body describes this spec-143 recurrence, the `MultiAddress::Index` width collision, the exact-fit bypass, the nested executable substitution, or demonstrated theft.

## Supporting consumer appendix: cold-wallet-app

Cold-wallet-app is not the report target. It is retained as a second consumer of the vulnerable SDK decoder and as supporting evidence that original bytes can be signed without canonical re-encoding. At app revision `d5fc4d65`, its bundled decoder is spec 136 while the chain revision's `Runtime::metadata()` reports spec 143.

The consumer renders a generic mismatch warning at `cold-wallet-app/lib/screens/sign_transaction_screen.dart:156-174`, but its Sign button is gated by scroll position rather than runtime equality at `cold-wallet-app/lib/screens/sign_transaction_screen.dart:227-255`. It signs the original payload bytes at `cold-wallet-app/lib/screens/sign_transaction_screen.dart:97-103`.

This supporting path requires a compromised online payload generator and a victim who signs after the generic warning. Those are not preconditions of the primary mobile multisig path. The warning says labels or parameters may be stale; it does not identify hidden `transfer_all` execution or say the account may be drained.

## Link to Proof of Concept

<https://gist.github.com/evolutionstorm/147a8c3695c6bfb3c7f8f80eebae6b44>

## Proof of Concept

The PoC files are attached to a GitHub Gist.

This PoC feeds one byte-identical call to the frozen generated spec-136 Quantus SDK codec and the frozen spec-143 Rust runtime. The SDK displays an indexed transfer plus a remark. The current runtime decodes a failed indexed transfer followed by transfer\_all to a different account. The listed mobile app's actual multisig confirmation sheet is exercised in a separate widget regression. Two independently executable wrappers are retained: force\_batch continues after a failed child, and if\_else dispatches its fallback after its main child fails.

### Listed mobile-app approval regression

Place `multisig_approval_regression_test.dart` in the frozen Quantus Apps mobile-app package root, then run:

```shell
: > .env
flutter pub get
flutter test multisig_approval_regression_test.dart
```

The test imports the real `MultisigActionConfirmSheet` and `CallDecoder`, loads the exact witness as authoritative proposal bytes, and proves the benign display, absence of a mismatch warning, enabled approval, and unchanged byte handoff to submission. See `mobile_app_README.md` for the assertions.

### `force_batch` witness

```
09040802000104020400d0000074b3aed30cdbb7c86d2da943ee944a3ee82ef086e6dfddc861f5df771600
```

Both call decoders consume all 43 bytes. The same valid signed extensions can therefore follow the call on both sides.

### `if_else` witness

```
090602000104020400d0000074b3aed30cdbb7c86d2da943ee944a3ee82ef086e6dfddc861f5df771600
```

Both call decoders consume all 42 bytes. Spec 136 displays the same indexed transfer as the main call and a 29-byte remark as the fallback. Spec 143 executes the failed `Index(())` transfer as the main call and decodes the old amount/remark bytes as a `transfer_all` fallback.

### Actual spec-136 Dart decoder

Use Dart 3.8 or newer:

```shell
dart pub get
dart run decode_spec136.dart
dart run decode_spec136.dart \
  090602000104020400d0000074b3aed30cdbb7c86d2da943ee944a3ee82ef086e6dfddc861f5df771600
```

Expected semantic tree:

```
Utility.force_batch([
  Balances.transfer_allow_death(Index(1), 872415488),
  System.remark(29 opaque bytes)
])

Utility.if_else(
  main=Balances.transfer_allow_death(Index(1), 872415488),
  fallback=System.remark(29 opaque bytes)
)
```

### Frozen current-runtime decoder, signed execution, and multisig execution

```shell
SKIP_WASM_BUILD=1 cargo test --offline -- --nocapture
```

The decoder tests assert both current call trees. Two execution tests sign each call with ML-DSA-65, submit it through `Executive::apply_extrinsic`, observe the wrapper's expected failure/fallback event, and prove `Balances::transfer_all` moved the victim's balance. A fifth regression creates a funded 2-of-2 multisig, proposes the attacker-supplied witness, asserts byte-identical storage, approves the stored bytes as the second signer, and executes them as the shared account. It proves the same nested witness drains the multisig balance. The retained destination was found by `grind_destination.rs`; its ML-DSA-65 seed is embedded in the test. Both force-batch paths then sign as that destination and transfer the received balance out to a third account, proving attacker control rather than only a keyless freeze.

### Bounded address search

```shell
cargo run --release --offline --bin grind_destination -- 50000000 16
```

The retained run found counter 816891 after 820391 attempted keys in 31.48 seconds.

Retained outputs are `spec136_force_batch.json`, `spec136_if_else.json`, `grind_output.txt`, and `test_output.txt`.
