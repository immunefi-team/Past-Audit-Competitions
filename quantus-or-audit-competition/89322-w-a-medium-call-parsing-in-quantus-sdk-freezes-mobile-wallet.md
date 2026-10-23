# 89322 w a medium call parsing in quantus sdk freezes mobile wallet&#x20;

**Submitted on Aug 18th 2026 at 15:24:16 UTC by @quasar for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89322
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

The shared call decoder in `quantus_sdk` describes the `main` sub-call of `utility.if_else` twice, so a chain of N nested `if_else` calls costs 2^N `describe()` invocations. A 230-byte payload freezes any consumer of the decoder, including the mobile wallet's multisig proposal sheet.

## Vulnerability Details

```dart
      case utility.IfElse(:final main, :final fallback):   // (A)
        return DecodedCall(
          pallet: 'Utility',
          call: 'if_else',
          fields: [
            NestedCallField('Primary call', describe(main)),   // (B)
            NestedCallField('Fallback call', describe(fallback)),
          ],
          summary: describe(main).summary,   // (C)
        );
```

* (A) `quantus_sdk/lib/src/chain/call_decoder.dart:572`: the `describe()` recursion has no depth counter, so nesting is unbounded.
* (B) First of the two traversals of the `main` subtree.
* (C) The same subtree is traversed a second time just to read `summary`, so each level doubles the work: T(N) = 2\*T(N-1) + O(1).

```dart
  static DecodedCall decodeBytes(List<int> bytes) {
    ...
    return describe(call);   // (D)
  }
```

* (D) `call_decoder.dart:59`: the multisig sheet feeds stored proposal bytes into this sink, where the linear codec decode hands off to the exponential `describe()`.

## Attack path

{% stepper %}
{% step %}

## Submit a nested call

A multisig co-signer submits `Multisig.propose` whose inner call is N=30 nested `utility.if_else` wrappers (pallet 9, call 6) around a `system.remark` leaf. The call encodes to 153 bytes: 30 levels at 5 bytes each (2 variant bytes plus the 3-byte fallback leaf), plus the 3-byte innermost leaf.
{% endstep %}

{% step %}

## Store the proposal

The chain stores the proposal; execution is not required for the bug.
{% endstep %}

{% step %}

## Open the proposal

The victim opens the proposal in the mobile app. `multisig_action_confirm_sheet.dart:189` runs `CallDecoder.decodeBytes(bytes)` on the stored call bytes.
{% endstep %}

{% step %}

## Trigger exponential parsing

`describe()` performs 2^30 = 1,073,741,824 invocations for a 153-byte call.
{% endstep %}

{% step %}

## Freeze the application

The sheet decodes synchronously on the UI isolate during build (`multisig_action_confirm_sheet.dart:353`). The app stops responding; reopening the proposal repeats the freeze as long as the proposal exists on chain.
{% endstep %}

{% step %}

## Reach the same sink through signing payloads

`QuantusPayloadParser.parsePayload` reaches the same `describe()` sink at `quantus_payload_parser.dart:159` on attacker-supplied signing payloads.
{% endstep %}

{% step %}

## Bypass the payload-size cap

The parser's size cap is 8192 bytes (`maxPayloadBytes`), which admits \~1600 nesting levels. A full signing payload is 230 bytes (153-byte call plus the 77-byte extensions suffix), and depth 30 doesn't finish.
{% endstep %}
{% endstepper %}

## Impact Details

A multisig co-signer freezes the mobile wallet of every member who opens the malicious proposal, and the freeze repeats on every open for as long as the proposal exists on chain.

## Proof of Concept

Get poc.txt, gist link - <https://gist.github.com/lgprbs/3b5c53cb748c746972267db996498f45>

Unpack it:

```bash
base64 -d poc.txt > poc.tgz && tar zxf poc.tgz
```

## How to repro

{% stepper %}
{% step %}

## Enter the packaged PoC folder

The Dockerfile pins the frozen program snapshot `d5fc4d6` and adds the test file, so the suite runs on a clean checkout.
{% endstep %}

{% step %}

## Build and run

```bash
$ docker build -t quantus-ifelse-dos .
$ docker run --rm quantus-ifelse-dos
```

{% endstep %}

{% step %}

## Verify the expected output

The same-size `batch_all` control finishes in about a second; both PoC tests pass only after their 15-second kill budget expires, and the suite ends with `All tests passed!`:

```
sizes: attack payloads are tiny ... +1
control: 30-deep batch_all chain parses quickly ... +2
PoC A-1: 30-deep if_else chain (~160-byte call) never finishes parsing ... +3 (after 15s)
PoC A-2: mobile-app path CallDecoder.decodeBytes also freezes (30-deep if_else) ... +4 (after 15s)
probe: 2000-deep batch_all (~8KB) decode path outcome ... deep batch_all 2000: crashed ... +5
All tests passed!
```

{% endstep %}

{% step %}

## Verify payload construction

The test file builds payloads with the repo's own generated runtime codecs, so every payload is encodable on chain as-is.
{% endstep %}
{% endstepper %}
