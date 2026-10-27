# 88243 bc insight check wormhole path accepts any path shape and hdwallet exports no path constructor so the node cli and the mobile sdk derive different wormhole keys for the same index

**Submitted on Aug 12th 2026 at 11:07:16 UTC by @Lun3x for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88243
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-qp-rusty-crystals/tree/audit-comp-ready/hdwallet>
* **Impacts:**

## Description

### Brief/Intro

The node CLI and the mobile SDK derive wormhole keys from different levels of the BIP-44 path for the same mnemonic and the same index. The CLI varies the account level (`m/44'/189189189'/{index}'/0'/0'`), the SDK varies the address-index level (`m/44'/189189189'/0'/0'/{index}'`), and the two only coincide at index 0. Mining rewards paid to a wormhole address derived by the CLI at any non-zero wallet index are therefore unreachable from the wallet app, whose mining-reward and redeem screens call the derivation with no index at all and so only ever look at index 0. The dilithium paths do not have this problem — both components vary the account level for those — which is what makes the wormhole case an oversight rather than a convention.

### Vulnerability Details

The CLI builds both paths the same way, varying the third segment. `chain/node/src/command.rs:109`:

```rust
let path =
    format!("m/44'/{QUANTUS_DILITHIUM_CHAIN_ID}/{index}'/0'/0'", index = wallet_index);
```

and `chain/node/src/command.rs:226`:

```rust
let path =
    format!("m/44'/{QUANTUS_WORMHOLE_CHAIN_ID}/{index}'/0'/0'", index = wallet_index);
```

So for `--wallet-index 1` the CLI's wormhole path is `m/44'/189189189'/1'/0'/0'`.

The SDK builds them differently from each other. `apps/quantus_sdk/lib/src/services/hd_wallet_service.dart:53-56` and `:67-77`:

```dart
Keypair _deriveHDWallet({required String mnemonic, int account = 0, int change = 0, int addressIndex = 0}) {
  final derivationPath = "m/44'/189189'/$account'/$change'/$addressIndex'";
  ...
}

Keypair keyPairAtIndex(String mnemonic, int index) {
  ...
  return _deriveHDWallet(mnemonic: mnemonic, account: index);      // dilithium: varies account
}

crypto.WormholeResult _deriveWormhole(String mnemonic, {int account = 0, int change = 0, int addressIndex = 0}) {
  final path = "m/44'/189189189'/$account'/$change'/$addressIndex'";
  ...
}

WormholeKeyPair deriveWormholeKeyPair({required String mnemonic, int index = 0}) =>
    WormholeKeyPair.fromResult(_deriveWormhole(mnemonic, addressIndex: index));   // wormhole: varies addressIndex
```

Every wormhole call site in the SDK passes `addressIndex`; none passes `account`. The change-address variant at `:76-77` also passes `change: 1, addressIndex: index`. So the SDK's wormhole index lives at segment 5 while the CLI's lives at segment 3.

Side by side, for the same mnemonic and index 1:

| Component                                | Dilithium                | Wormhole                    |
| ---------------------------------------- | ------------------------ | --------------------------- |
| CLI (`command.rs:109`, `:226`)           | `m/44'/189189'/1'/0'/0'` | `m/44'/189189189'/1'/0'/0'` |
| SDK (`hd_wallet_service.dart:54`, `:68`) | `m/44'/189189'/1'/0'/0'` | `m/44'/189189189'/0'/0'/1'` |

The dilithium row agrees. The wormhole row does not, and because BIP-32 derivation is hardened at every level here, the two wormhole paths produce unrelated keys and therefore unrelated addresses. Only index 0 coincides, where both reduce to `m/44'/189189189'/0'/0'/0'`.

The consequence is concrete in the app's mining-reward flow, which does not even expose an index. `mobile-app/lib/providers/mining_rewards_provider.dart:29`:

```dart
final keyPair = ref.watch(hdWalletServiceProvider).deriveWormholeKeyPair(mnemonic: mnemonic);
```

and `mobile-app/lib/v2/screens/settings/redeem_progress_screen.dart:48`:

```dart
final keyPair = ref.read(hdWalletServiceProvider).deriveWormholeKeyPair(mnemonic: mnemonic);
```

Both omit `index`, so both default to `index: 0` and look only at `m/44'/189189189'/0'/0'/0'`. A miner who ran the node with `--wallet-index 1` accrued rewards at `m/44'/189189189'/1'/0'/0'`. Restoring that same mnemonic in the wallet shows nothing for those rewards and the redeem screen cannot act on them, because the app never derives that path — not at index 0, and not at any index it can be given, since its index moves a different segment.

The library that both sides call does not constrain this. `qp-rusty-crystals-hdwallet`'s `check_wormhole_path` (`hdwallet/src/lib.rs:421-427`) validates only the coin type in segment 2:

```rust
fn check_wormhole_path(path: &str) -> Result<(), HDLatticeError> {
    check_derivation_path(path)?;
    if path.split("/").nth(2) != Some(QUANTUS_WORMHOLE_CHAIN_ID) {
        return Err(HDLatticeError::InvalidWormholePath(path.to_string()));
    }
    Ok(())
}
```

Any depth and any arrangement of the remaining segments is accepted, and the crate exports the coin-type constants but no path constructor, so each consumer invents its own layout. That is why two first-party components diverged without either being wrong by the library's contract.

### Impact Details

The in-scope impact this maps to, quoted verbatim from the program page:

> Security Best practices

Funds are not lost and nothing is stolen. Mining rewards accrued at a non-zero CLI wallet index remain fully spendable by the CLI, which derives the same path it wrote. What is lost is discoverability through the wallet: the app cannot display or redeem them, and a user has no indication that the balance exists or that a path convention is the reason. Recovering them requires diagnosing that two first-party tools index different BIP-44 levels, which is not something a user can be expected to work out.

The stronger lines do not apply and the report does not claim them. This is not "Permanent freezing of funds", because the CLI recovers the balance without any code change. It is not a Low-tier fee or weight inaccuracy either. It is an interoperability defect between two in-scope components with a user-visible consequence, which is why it is filed as an Insight.

The fix is to pick one convention and export it from the library that both sides already depend on: add a path constructor to `qp-rusty-crystals-hdwallet` alongside the existing coin-type constants, have `check_wormhole_path` enforce the full expected shape rather than only segment 2, and change whichever side is non-conforming. Given the dilithium paths already agree on the account level, aligning the SDK's wormhole derivation to vary `account` is the smaller change and makes the two families internally consistent as well.

## References

* `chain/node/src/command.rs:109` — CLI dilithium path, varies the account segment
* `chain/node/src/command.rs:226` — CLI wormhole path, varies the account segment
* `apps/quantus_sdk/lib/src/services/hd_wallet_service.dart:53-56, 58-64` — SDK dilithium path, varies `account`
* `apps/quantus_sdk/lib/src/services/hd_wallet_service.dart:67-77` — SDK wormhole path, varies `addressIndex`
* `mobile-app/lib/providers/mining_rewards_provider.dart:29` — mining-reward derivation with no index, so index 0 only
* `mobile-app/lib/v2/screens/settings/redeem_progress_screen.dart:48` — redeem flow, same
* `qp-rusty-crystals/hdwallet/src/lib.rs:421-427` — `check_wormhole_path` validates only the coin type
* `qp-rusty-crystals/hdwallet/src/lib.rs:85-88` — coin-type constants exported with no path constructor
* Asset revisions: `audit-comp-quantus-chain` @ `3b243b870a5442a6e5f443056f67bbba87d472f5`; `audit-comp-quantus-apps` @ `d5fc4d6572ebc2244e95009a33f5da5048935867`; `audit-comp-quantus-qp-rusty-crystals` @ `94dfe6e671b29689c66148991630406e983e2457`

## Proof of Concept

The defect is a path-construction mismatch between two in-scope components, so it is demonstrated by reading the two path expressions and showing the derived keys differ. Both are single expressions in first-party source; no exploit is involved and nothing touches mainnet or any public testnet. This is a Substrate node plus a Dart SDK, so the "fork mainnet with Hardhat/Foundry" instruction does not apply.

{% stepper %}
{% step %}

## The two path expressions, verbatim

```bash
sed -n '107,110p;225,227p' chain/node/src/command.rs
```

```rust
// Build the derivation path (all components must be hardened for lattice-based crypto)
let path =
    format!("m/44'/{QUANTUS_DILITHIUM_CHAIN_ID}/{index}'/0'/0'", index = wallet_index);
...
let path =
    format!("m/44'/{QUANTUS_WORMHOLE_CHAIN_ID}/{index}'/0'/0'", index = wallet_index);
```

```bash
sed -n '53,56p;67,77p' apps/quantus_sdk/lib/src/services/hd_wallet_service.dart
```

```dart
Keypair _deriveHDWallet({required String mnemonic, int account = 0, int change = 0, int addressIndex = 0}) {
  final derivationPath = "m/44'/189189'/$account'/$change'/$addressIndex'";
  return crypto.generateDerivedKeypair(mnemonicStr: mnemonic, path: derivationPath);
}
...
crypto.WormholeResult _deriveWormhole(String mnemonic, {int account = 0, int change = 0, int addressIndex = 0}) {
  final path = "m/44'/189189189'/$account'/$change'/$addressIndex'";
  return crypto.deriveWormhole(mnemonicStr: mnemonic, path: path);
}

/// Derive the wormhole key pair at HD index `index` (account=0, change=0).
WormholeKeyPair deriveWormholeKeyPair({required String mnemonic, int index = 0}) =>
    WormholeKeyPair.fromResult(_deriveWormhole(mnemonic, addressIndex: index));
```

Substituting `index = 1`:

```
CLI  dilithium : m/44'/189189'/1'/0'/0'
SDK  dilithium : m/44'/189189'/1'/0'/0'      <-- agree

CLI  wormhole  : m/44'/189189189'/1'/0'/0'
SDK  wormhole  : m/44'/189189189'/0'/0'/1'   <-- differ
```

Every segment is hardened, so these two wormhole paths derive independent keys with independent addresses. They coincide only at `index = 0`.
{% endstep %}

{% step %}

## Confirm the SDK never varies the account level for wormhole

```bash
grep -n "_deriveWormhole\|deriveWormholeKeyPair" apps/quantus_sdk/lib/src/services/hd_wallet_service.dart
```

```
67:  crypto.WormholeResult _deriveWormhole(String mnemonic, {int account = 0, int change = 0, int addressIndex = 0}) {
69:    return crypto.deriveWormhole(mnemonicStr: mnemonic, path: path);
73:  WormholeKeyPair deriveWormholeKeyPair({required String mnemonic, int index = 0}) =>
74:      WormholeKeyPair.fromResult(_deriveWormhole(mnemonic, addressIndex: index));
77:      WormholeKeyPair.fromResult(_deriveWormhole(mnemonic, change: 1, addressIndex: index));
```

Two call sites, both passing `addressIndex`. Neither passes `account`. There is no SDK entry point that produces the CLI's layout.
{% endstep %}

{% step %}

## The app's mining-reward path is index 0 only

```bash
grep -rn "deriveWormholeKeyPair" --include='*.dart' mobile-app/
```

```
mobile-app/lib/providers/mining_rewards_provider.dart:29:  final keyPair = ref.watch(hdWalletServiceProvider).deriveWormholeKeyPair(mnemonic: mnemonic);
mobile-app/lib/v2/screens/settings/redeem_progress_screen.dart:48:      final keyPair = ref.read(hdWalletServiceProvider).deriveWormholeKeyPair(mnemonic: mnemonic);
```

Both omit `index`, so both take the `index = 0` default and derive `m/44'/189189189'/0'/0'/0'`.
{% endstep %}
{% endstepper %}

### Reproducing the divergence in derived output

Because both sides ultimately call the same Rust library, the mismatch can be shown directly against `qp-rusty-crystals-hdwallet` without a Flutter toolchain. `Cargo.toml`:

```toml
[package]
name = "pathdiv"
version = "0.1.0"
edition = "2021"

[dependencies]
qp-rusty-crystals-hdwallet = { path = "./repos/qp-rusty-crystals/hdwallet" }

[workspace]
```

`src/main.rs`:

```rust
use qp_rusty_crystals_hdwallet::generate_wormhole_from_mnemonic;

fn main() {
    // Any valid BIP-39 mnemonic; the point is that one mnemonic yields two addresses.
    let m = "abandon abandon abandon abandon abandon abandon \
             abandon abandon abandon abandon abandon about";

    let cli_path = "m/44'/189189189'/1'/0'/0'";   // chain/node/src/command.rs:226, wallet_index = 1
    let sdk_path = "m/44'/189189189'/0'/0'/1'";   // hd_wallet_service.dart:68, index = 1

    for (label, p) in [("CLI  (varies account)", cli_path), ("SDK  (varies addressIndex)", sdk_path)] {
        match generate_wormhole_from_mnemonic(m, None, p) {
            Ok(pair) => println!("{label}  {p}\n    address = {}", hex::encode(pair.address)),
            Err(e) => println!("{label}  {p}\n    error: {e:?}"),
        }
    }
    println!("\nBoth paths pass check_wormhole_path (segment 2 is the wormhole coin type),");
    println!("and both are accepted by the library, but they are different keys.");
}
```

`cargo run --release` prints two different addresses for the same mnemonic and the same user-facing index of 1. Index 0 is the only value for which the two expressions coincide.

### Conditions under which this insight is valuable

* A miner who runs the node with `--wallet-index 1` or higher accrues mining rewards at a wormhole address the wallet app can never derive. The app's mining-reward provider and redeem screen pass no index at all, so they look only at index 0.
* The funds are not lost: the CLI derives the same path it wrote and can still spend them. The loss is discoverability, and the user has no signal that a balance exists or that a path convention is the cause.
* The failure is silent in both directions. Neither component warns, because both paths are valid per `check_wormhole_path`, which checks only that segment 2 is the wormhole coin type (`hdwallet/src/lib.rs:421-427`).
* The dilithium paths agree between the two components, so a user who verifies their normal account matches across CLI and app gains false confidence that the wormhole side matches too.

### Scope of this measurement

This shows the two path expressions differ and that the library accepts both. It does not involve a running app: there is no Flutter toolchain in this environment, so the SDK side is established from its source expression and its call sites rather than by executing the Dart. The Rust harness above derives both paths through the same library both components use, which is the shared component where the divergence becomes two distinct addresses.

### Remediation

Export a single path constructor from `qp-rusty-crystals-hdwallet` next to the existing `QUANTUS_WORMHOLE_CHAIN_ID` / `QUANTUS_DILITHIUM_CHAIN_ID` constants, have `check_wormhole_path` enforce the full expected shape rather than only segment 2, and change whichever consumer does not conform. Aligning the SDK to vary the account segment is the smaller change and also makes its wormhole derivation consistent with its own dilithium derivation.

Regression property: for a fixed mnemonic and each index in `0..n`, the wormhole address produced by the CLI path expression and by the SDK path expression must be equal. That test fails today for every index except 0.
