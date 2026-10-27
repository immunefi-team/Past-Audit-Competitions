# 88382 w a insight wallet mnemonic stored with default ios keychain accessibility ksecattraccessiblewhenunlocked seed is backup extractable and not device bound

**Submitted on Aug 12th 2026 at 23:26:07 UTC by @promael for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88382
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**

## Description

## Brief/Intro

The Quantus SDK persists the wallet's BIP39 mnemonic — the single seed from which every account key (including Dilithium/ML-DSA signing keys and wormhole secrets) is derived — using `FlutterSecureStorage` with no platform accessibility options set. On iOS this falls back to the library default `kSecAttrAccessibleWhenUnlocked`, which stores the item **without the `ThisDeviceOnly` protection class**. Such Keychain items are **included in encrypted device backups** and are **migratable to a different device** on restore. As a result, an attacker who obtains a victim's encrypted iTunes/Finder backup (and its backup password) can extract the plaintext seed phrase from the backup and restore full wallet control onto their own device — a path that a `…ThisDeviceOnly` protection class would close entirely. This is a defense-in-depth gap for the most sensitive secret in the system, not a remotely triggerable vulnerability.

## Vulnerability Details

The only `FlutterSecureStorage` instance in the in-scope code is constructed with no `IOSOptions`/`AndroidOptions`:

```dart
// quantus_sdk/lib/src/services/settings_service.dart:16
final _secureStorage = const FlutterSecureStorage(
  mOptions: MacOsOptions(usesDataProtectionKeychain: false),
);
```

The wallet mnemonic is written to and read from this store in plaintext:

```dart
// settings_service.dart:349-358
String getMnemonicKey(int walletIndex) =>
    walletIndex == 0 ? 'mnemonic' : 'mnemonic_$walletIndex';

Future<void> setMnemonic(String mnemonic, int walletIndex) async {
  await _secureStorage.write(key: getMnemonicKey(walletIndex), value: mnemonic);
}

Future<String?> getMnemonic(int walletIndex) async {
  return await _secureStorage.read(key: getMnemonicKey(walletIndex));
}
```

`flutter_secure_storage` (pinned `^10.1.0`) defaults, when no `IOSOptions(accessibility: …)` is provided, to `KeychainAccessibility.unlocked` → the iOS Keychain attribute **`kSecAttrAccessibleWhenUnlocked`**. Two consequences follow from that protection class:

1. **Included in encrypted backups.** Items with `kSecAttrAccessibleWhenUnlocked` are copied into an *encrypted* iTunes/Finder backup and, on restore, are re-installed onto the target device. Only the `…ThisDeviceOnly` variants (`kSecAttrAccessibleWhenUnlockedThisDeviceOnly` / `…AfterFirstUnlockThisDeviceOnly`) are excluded from every backup and bound to the originating device's hardware.
2. **Not device-bound.** Because the item is not `ThisDeviceOnly`, the seed can be materialized on hardware other than the one that generated it.

A grep across the two in-scope Dart trees (`quantus_sdk/lib`, `mobile-app/lib`) confirms **zero** occurrences of `IOSOptions`, `AndroidOptions`, `KeychainAccessibility`, or `accessibility` — i.e. the accessibility class is never overridden anywhere, and this single store holds the mnemonic for every wallet index.

Scope note (honest): this is an **iOS/macOS** issue. On Android, `flutter_secure_storage` encrypts values with a key held in the Android Keystore, whose master key is non-exportable; even if the encrypted blob is captured by Android auto-backup, it cannot be decrypted off-device, so the Android default is materially safer. The additional `usesDataProtectionKeychain: false` on macOS selects the weaker legacy file-based keychain over the modern data-protection keychain — a secondary hardening gap on macOS.

## Impact Details

The mnemonic is the root secret controlling all funds (it deterministically derives the Dilithium account keys and the `m/44'/189189189'/…` wormhole secrets). With the default protection class, the attack path is:

{% stepper %}
{% step %}

## Obtain an encrypted backup

Attacker obtains the victim's **encrypted** local device backup (e.g. an unattended computer holding a Finder/iTunes backup, a compromised backup archive, or a forensic image) **and** the backup password.
{% endstep %}

{% step %}

## Extract the mnemonic

The attacker extracts the Keychain item `mnemonic` from the backup (standard backup-parsing tooling recovers `WhenUnlocked` Keychain items from an encrypted backup) — obtaining the **plaintext seed phrase**.
{% endstep %}

{% step %}

## Import and drain funds

The attacker imports the seed into any Quantus wallet and drains all funds.
{% endstep %}
{% endstepper %}

The `ThisDeviceOnly` protection class would make steps 1–2 impossible (the item is never present in any backup and never leaves the device). Because exploitation requires possession of an encrypted backup plus its password — not a remote or on-chain trigger — this is honestly an **Insight / Low** (best-practice hardening for key material), not a High/Critical. It is nonetheless a real, end-to-end-traceable weakness in how the most sensitive secret is stored, and it is trivially fixable.

## Recommended fix

Pin the strongest protection class for the seed store (and use the modern data-protection keychain on macOS):

```dart
final _secureStorage = const FlutterSecureStorage(
  iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock_this_device),
  mOptions: MacOsOptions(
    accessibility: KeychainAccessibility.first_unlock_this_device,
    usesDataProtectionKeychain: true,
  ),
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
);
```

`first_unlock_this_device` (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`) keeps the seed usable for background operations while excluding it from all backups and binding it to the device. If background access is not required, `unlocked_this_device` is even stricter.

## References

* `quantus_sdk/lib/src/services/settings_service.dart:16` (storage construction, no accessibility options)
* `quantus_sdk/lib/src/services/settings_service.dart:352-358` (`setMnemonic`/`getMnemonic` — plaintext seed write/read)
* `quantus_sdk/pubspec.yaml:36` (`flutter_secure_storage: ^10.1.0`)
* Apple, *Keychain Item Accessibility Constants* — `kSecAttrAccessibleWhenUnlocked` vs `…ThisDeviceOnly` (backup inclusion / device binding semantics)

## Proof of Concept

### Proof of Concept (step-by-step; no runnable exploit required for an Insight)

Conditions under which the weakness is valuable / demonstrable:

{% stepper %}
{% step %}

## Create or import a wallet

Build and run the in-scope app on an iOS device; create or import a wallet so `setMnemonic` writes the mnemonic to the Keychain via the `settings_service.dart:16` store (no accessibility override → `kSecAttrAccessibleWhenUnlocked`).
{% endstep %}

{% step %}

## Create an encrypted backup

Create an **encrypted** local backup of the device (Finder/iTunes "Encrypt local backup", with a password).
{% endstep %}

{% step %}

## Parse the Keychain item

Parse the encrypted backup's Keychain with any standard backup-inspection tool. The item with account key `mnemonic` is present (it would be **absent** had the store used a `…ThisDeviceOnly` accessibility class), and its value is the plaintext BIP39 seed phrase.
{% endstep %}

{% step %}

## Import the seed

Import that seed into a fresh Quantus wallet on a different device → full control of the victim's funds, demonstrating the seed is neither backup-excluded nor device-bound.
{% endstep %}
{% endstepper %}

Contrast: after applying the recommended `first_unlock_this_device` fix and repeating steps 1–3, the `mnemonic` item no longer appears in the encrypted backup — closing the extraction path.
