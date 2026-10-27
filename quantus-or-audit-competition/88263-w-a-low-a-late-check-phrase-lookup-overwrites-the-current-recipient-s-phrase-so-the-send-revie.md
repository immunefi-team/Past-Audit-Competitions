# 88263 w a low a late check phrase lookup overwrites the current recipient s phrase so the send review screen vouches for an address the user is not paying

**Submitted on Aug 12th 2026 at 14:18:29 UTC by @koshak01 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88263
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions

## Description

**Target:** `immunefi-team/audit-comp-quantus-apps`, branch `audit-comp-ready`, commit `d5fc4d6572ebc2244e95009a33f5da5048935867` -- `mobile-app` (defect), `quantus_sdk` (the check-phrase service it calls).

## Brief / Intro

The wallet's defence against address substitution is a human-readable check phrase: the user compares a short phrase shown next to the recipient with the phrase the payee reads out. The phrase is computed asynchronously, in a background isolate, with a per-address cache.

In `SelectRecipientScreen._lookupAddress` the lookup result is written to state **unconditionally** -- there is no check that the answer still belongs to the address currently in the field. When two lookups are in flight and the slower one is for the *previous* address, its late answer overwrites the phrase of the address the user actually kept.

The stale value is then **trusted for the rest of the flow**: the amount screen keeps a phrase it was handed instead of recomputing it, so the final review screen -- the last screen before signing -- displays a phrase belonging to a different address than the one that will be paid.

## Vulnerability Details

### 1. The missing guard, next to an identical call that has one

`mobile-app/lib/v2/screens/send/select_recipient_screen.dart`, both statements inside the same method:

```dart
// line 122-126 -- self-send probe: guarded
widget.strategy.isSelfRecipient(ref, address).then((isSelf) {
  if (!mounted || _recipientController.text.trim() != address) return;   // <-- guard
  ...
});

// line 138-140 -- check phrase: unguarded
checksumService.getHumanReadableName(address).then((checksum) {
  if (mounted) setState(() => _recipientChecksum = checksum);            // <-- no guard
});
```

The same screen starts two asynchronous lookups for the same address. The first one refuses a late answer; the second one accepts it.

### 2. The stale value is propagated, not recomputed

`mobile-app/lib/v2/screens/send/input_amount_screen.dart:88-94`:

```dart
if (widget.recipientChecksum != null) {
  _recipientChecksum = widget.recipientChecksum;   // accepted as given
} else {
  _checksumService.getHumanReadableName(widget.recipientAddress.trim()).then(...);
}
```

A phrase handed in from the previous screen is never re-derived, so a wrong value cannot self-correct. It is passed on to the review screen and rendered next to the recipient (`review_send_screen.dart:199-203`).

### 3. Why the ordering is deterministic, not a lucky race

`quantus_sdk/lib/src/services/human_readable_checksum_service.dart:66-95` caches by address: a hit returns from the map before any isolate work, while a miss sends the address to a background isolate and awaits the reply over a fresh port (and, on first use, waits for isolate start-up). The asymmetry is therefore in the code path itself -- an early return against a cross-isolate round trip -- not an assumption about scheduler luck. The project's own unit test names this behaviour "serves repeat lookups from cache" (`test/unit/human_readable_checksum_service_test.dart`); that test asserts value equality, not latency, so the timing claim here rests on the code path above.

`select_recipient_screen.dart:68-90` (`_loadRecents`, invoked when the screen opens) pre-computes the phrase for **every** recent address, so every recent address is warm before the user touches anything.

That makes the harmful order reproducible rather than accidental:

{% stepper %}
{% step %}

### The screen opens

All recent addresses are cached.
{% endstep %}

{% step %}

### Enter a non-recent address

The user pastes or scans an address that is **not** in recents -- a cache miss, slow.
{% endstep %}

{% step %}

### Select a recent address

The user then taps a **recent** address -- a cache hit, fast; its phrase lands and is displayed.
{% endstep %}

{% step %}

### The stale response arrives

The slow answer for the pasted address arrives and overwrites the phrase.
{% endstep %}

{% step %}

### The address and phrase diverge

The field holds the recent address, while the phrase belongs to the pasted one.
{% endstep %}

{% step %}

### The incorrect phrase reaches review

The amount screen keeps that phrase, and the review screen shows it under the recipient.
{% endstep %}
{% endstepper %}

### 4. This is an oversight, not a deliberate trade-off

Three asynchronous values feed the same send flow, and two of them are already protected against staleness -- by two different mechanisms:

* **fee estimate:** a generation counter orphans in-flight fetches whenever the amount changes, with the rule spelled out in the code (`input_amount_screen.dart:145-155`): a stale fee "must never stay valid for a new amount. Drop it and orphan any in-flight fetch";
* **self-send probe:** the address guard quoted above (`select_recipient_screen.dart:125`);
* **check phrase:** nothing.

Nothing in the repository presents the third case as intended. There is no comment at the call site or on the service saying the phrase may lag the address; no test pins the current behaviour; and the commit that last touched this method changed a line thirteen lines away without adding the guard.

The guard pattern `_recipientController.text.trim() != address` occurs exactly once in the whole application -- at the sibling call in this same method. That is stated precisely rather than generously: this is not a house style applied everywhere but here; it is the one place where the hazard was handled, thirteen lines above the place where it was not.

## Impact Details

The check phrase exists to catch a substituted recipient. In the state above it does the opposite: it **confirms** an address the user is not verifying, on the last screen before the signature. A user who follows the intended procedure -- compare the phrase, then approve -- is led to approve a payment to a different address than the one the phrase belongs to.

The stale phrase is not confined to the software review screen: the same value is handed to the hardware-signing session as the third detail line shown to the user (`regular_send_strategy.dart:121,138`), and to the multisig proposal flow (`multisig_propose_strategy.dart:233,254`). Users who added an external signing device -- the most cautious group -- see the same wrong phrase on the device-review screen.

Scope of the claim, stated plainly:

* both address changes are made by the user; an attacker cannot trigger the second one remotely. The `/pay` deep link does not reach this screen at all -- it opens the amount screen directly and refuses to interrupt a send already in progress (`home_screen.dart:121-144`);
* the recents list is fed only by the user's own outgoing transfers (`regular_send_strategy.dart:160,179`, `multisig_propose_strategy.dart:224`, `encrypted_send_provider.dart:151`), so an attacker cannot plant an address there with an incoming dust transfer;
* the full address is displayed next to the phrase, so a user who reads all of it is not fooled.

This is therefore reported as a defeated safety control, not as direct theft.

## Recommendation

Apply, at the check-phrase call site, the guard that the sibling call in the same method already uses:

```dart
checksumService.getHumanReadableName(address).then((checksum) {
  if (!mounted || _recipientController.text.trim() != address) return;
  setState(() => _recipientChecksum = checksum);
});
```

The same one-line guard is needed in `redeem_address_screen.dart:73-75`, which repeats the pattern.

A stronger variant, already used elsewhere in this feature for fee estimates, is a generation counter that orphans in-flight lookups when the input changes (`input_amount_screen.dart:145-155`).

## Link to Proof of Concept

<https://gist.github.com/koshak01/f335bcead6be60007ee1b03fbbdb2b93>

## Proof of Concept

Reproduced by a widget test on the in-scope revision. No network, no chain, no device; the repository sources are not modified (the test is a new file).

**Command**

```bash
cd mobile-app && flutter test test/unit/recipient_checksum_race_test.dart
```

**Output (verbatim)**

```
00:00 +0: delayed checksum for A overwrites phrase while address B is shown
RESULT: shown address = "ADDRESS_B_qzABCDEF48YN9EQe2ito734iCReTXjnungmNCSY4Yph1YzXYZ" | shown phrase = "ALPHA-ANCHOR-APPLE-ARMOR"
00:00 +1: All tests passed!
```

`ALPHA-ANCHOR-APPLE-ARMOR` is the phrase of address **A**, while the field holds address **B**.

**What the test demonstrates directly, and what is not claimed from it.** The run above proves one thing: on the recipient screen, a late answer for a previous address overwrites the phrase while a different address is in the field. The onward propagation -- the amount screen keeping a phrase it was handed, the review screen and the hardware-signing session rendering it -- is established by reading the code paths cited in the Description, not by this test. No claim in this report goes beyond those two sources.

**Environment**

| Item                          | Value                                                              |
| ----------------------------- | ------------------------------------------------------------------ |
| Repository                    | `immunefi-team/audit-comp-quantus-apps`, branch `audit-comp-ready` |
| Commit (`git rev-parse HEAD`) | `d5fc4d6572ebc2244e95009a33f5da5048935867`                         |
| Flutter                       | 3.44.9 (`6b182d2c758`)                                             |
| Dart                          | 3.12.2 (stable)                                                    |

**How the test isolates the ordering.** The check-phrase service is replaced through the existing provider (`humanReadableChecksumServiceProvider`), so no production code is touched. The stand-in resolves address A only when the test says so, and returns address B immediately -- the same fast/slow asymmetry the real cache produces between a hit and a miss. The ordering is therefore fixed by the test, not by timing luck.

{% hint style="info" %}
For exact reproduction, `flutter test` fails out of the box with `No file or variants found for asset: .env`, because `pubspec.yaml` declares `.env` as an asset. Creating an empty `.env` in `mobile-app/` is sufficient; no source file is changed.
{% endhint %}

The test source: <https://gist.github.com/koshak01/f335bcead6be60007ee1b03fbbdb2b93>

Save it as `mobile-app/test/unit/recipient_checksum_race_test.dart` and run the command above.
