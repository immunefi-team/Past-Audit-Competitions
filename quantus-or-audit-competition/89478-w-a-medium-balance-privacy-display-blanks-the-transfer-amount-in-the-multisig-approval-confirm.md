# 89478 w a medium balance privacy display blanks the transfer amount in the multisig approval confirmation enabling blind co signer approval and direct theft of user funds

**Submitted on Aug 19th 2026 at 14:49:07 UTC by @zealynx for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89478
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions
  * Direct theft of user funds

## Description

### Brief/Intro

The Quantus mobile wallet renders the transfer amount in the multisig approval confirmation through the same display provider it uses for portfolio balances, so the user's balance-privacy toggle blanks it. When a user has enabled balance privacy, a native-token transfer nested in `Utility.batch` renders with its call name, destination, and structure intact, but its amount is replaced by the placeholder `-----`, while the **Approve** button stays enabled. A malicious co-signer of an ordinary funded multisig can exploit this to obtain an uninformed approval and drain the shared account — the confirmation never shows the amount the victim is signing. In production this causes direct theft of up to the full multisig balance; the proof of concept signs and executes an exact transfer of 1,000,000 QUAN to the attacker that was never displayed.

### Vulnerability Details

The confirmation amount crosses this production path:

| Location                                                                     | Behaviour                                                                                                                     |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `quantus_sdk/lib/src/services/settings_service.dart`                         | Persists the user-selected `balance_hidden` preference.                                                                       |
| `mobile-app/lib/providers/wallet_providers.dart` (`isBalanceHiddenProvider`) | Publishes the persisted preference.                                                                                           |
| `mobile-app/lib/providers/currency_display_provider.dart`                    | For balance-hidden state, replaces every transaction amount's display value with `-----`.                                     |
| `mobile-app/lib/v2/components/decoded_call_view.dart`                        | Renders the transfer parameter through that preference-dependent value.                                                       |
| `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart`      | Decodes the authoritative stored call bytes and renders their parameters through `DecodedCallView`, with **Approve** enabled. |

The confirmation decodes the authoritative on-chain bytes **correctly** — the wrapper `Utility · batch`, the inner `Balances · transfer_allow_death`, and the attacker's full destination address all render. Only the amount is dropped, and it is dropped by the display preference:

```dart
// mobile-app/lib/v2/components/decoded_call_view.dart  (AmountField case)
final value = assetId == null
    ? ref.watch(txAmountDisplayProvider)(token, isSend: true).primaryAmount // '-----' when balance-hidden
    : '$token (asset #$assetId, raw units)';
```

Balance privacy is a portfolio-display preference, not authorization to omit a transaction parameter from a signing confirmation. The sheet provides no amount-reveal control and no warning, and leaves **Approve** enabled.

#### Vulnerable scenario

{% stepper %}
{% step %}

### An attacker and a victim are the two members of an ordinary funded 2-of-2 multisig

The victim has enabled the released balance-privacy toggle — a production-reachable state selected from the home screen.
{% endstep %}

{% step %}

### The attacker proposes a transfer

`Utility.batch([ Balances.transfer_allow_death(attacker, 1_000_000 QUAN) ])`
{% endstep %}

{% step %}

### The victim opens the approval sheet

It shows `Utility · batch`, `Balances · transfer_allow_death`, and the attacker's destination address, but renders `Amount: -----`. **Approve** is enabled.
{% endstep %}

{% step %}

### The victim approves

The wallet signs the exact stored call bytes with the production ML-DSA key. Once the threshold is met the attacker executes; the runtime moves 1,000,000 QUAN from the multisig to the attacker.
{% endstep %}
{% endstepper %}

The attacker uses only ordinary signed multisig and Balances calls. No remote-data or exchange-rate manipulation, compromised infrastructure, victim key, or privileged chain origin is required, and the attacker cannot change the victim's preference.

**Recommended fix.** Render native amounts in signing confirmations independently of exchange-rate, fiat-first, and balance-privacy preferences. The exact chain-denominated amount must remain mandatory even when the user hides portfolio balances:

```diff
// mobile-app/lib/v2/components/decoded_call_view.dart
-            ? ref.watch(txAmountDisplayProvider)(token, isSend: true).primaryAmount
+            ? '-${ref.watch(numberFormattingServiceProvider).formatBalance(token, smartDecimals: 4, maxDecimals: AppConstants.decimals, addSymbol: true)}'
```

### Impact Details

A malicious co-signer obtains a victim's approval signature for a transfer whose amount was never shown, then executes it, stealing up to the entire funded multisig balance. This exceeds the co-signer's baseline powers: a 2-of-2 member cannot move shared funds alone — the exploit manufactures the missing second signature by hiding the amount, so the loss is theft, not merely the freeze a hostile co-signer could already cause.

The proof of concept demonstrates an exact loss of **1,000,000 QUAN** from the multisig with a matching attacker credit, proven end to end: the confirmation renders `Amount: -----`, the exact bytes passed to the ML-DSA signer independently decode to the 1,000,000 QUAN transfer, and the production runtime accepts the byte-identical extrinsic and moves the funds.

Selected in-scope Websites & Apps impacts:

* **Malicious interactions with an already-connected wallet** — the confirmation omits a transaction parameter (the amount) while leaving Approve enabled, inducing the victim to submit a malicious transaction.
* **Direct theft of user funds** — the demonstrated consequence (1,000,000 QUAN extracted).

## References

Frozen revisions: `quantus-apps @ d5fc4d6572ebc2244e95009a33f5da5048935867` · `quantus-chain @ 3b243b870a5442a6e5f443056f67bbba87d472f5`

* <https://github.com/immunefi-team/audit-comp-quantus-apps/blob/audit-comp-ready/mobile-app/lib/v2/components/decoded\\_call\\_view.dart#L66-L83>
* <https://github.com/immunefi-team/audit-comp-quantus-apps/blob/audit-comp-ready/mobile-app/lib/providers/currency\\_display\\_provider.dart>
* <https://github.com/immunefi-team/audit-comp-quantus-apps/blob/audit-comp-ready/mobile-app/lib/providers/wallet\\_providers.dart>
* <https://github.com/immunefi-team/audit-comp-quantus-apps/blob/audit-comp-ready/mobile-app/lib/v2/screens/multisig/multisig\\_action\\_confirm\\_sheet.dart>
* <https://github.com/immunefi-team/audit-comp-quantus-apps/blob/audit-comp-ready/quantus\\_sdk/lib/src/services/settings\\_service.dart>

## Link to Proof of Concept

<https://gist.github.com/vendrell46/f28de277cdd4571e5edaf4207dfe6d73>

## Proof of Concept

Frozen revisions: `quantus-apps @ d5fc4d6` · `quantus-chain @ 3b243b8`. Two halves prove the full path: the wallet hides the amount and signs the real bytes, and the production runtime accepts those exact bytes and moves the funds. The complete test files are in the secret gist linked at the end; the steps and observed output below make the report reproducible on its own.

### Step by step

{% stepper %}
{% step %}

### Mobile half (`privacy_batch_confirmation_poc_test.dart`)

Enables balance privacy through the production `SettingsService().setBalanceHidden(true)` path, builds `Utility.batch([ transfer_allow_death(attacker, 1,000,000 QUAN) ])`, and renders the real `MultisigActionConfirmSheet` with an honest 1:1 exchange rate.

It captures every rendered `Text`: the sheet shows `Utility · batch`, `Balances · transfer_allow_death`, the attacker destination, and `Amount: -----`, with **Approve** enabled. It taps Approve, constructs the production signing payload, signs it with the real Rust ML-DSA FFI, verifies the signature, and independently SCALE-decodes the signed bytes back to the 1,000,000 QUAN transfer to the attacker.
{% endstep %}

{% step %}

### Chain half (`runtime_fiat_batch_acceptance.rs`)

Reconstructs the byte-identical signing payload with the production runtime and full `TxExtension` tuple, asserts it equals the bytes captured at the mobile ML-DSA boundary, applies the signed approval, executes the proposal, and asserts the multisig free balance falls by exactly 1,000,000 QUAN while the attacker's rises by the same amount (plus the returned proposal deposit).
{% endstep %}
{% endstepper %}

### Run

```sh
# mobile half
cd quantus-apps/quantus_sdk && ./test/build_library.sh
cd ../mobile-app && test -e .env || touch .env
flutter test --no-pub test/privacy_batch_confirmation_poc_test.dart --tags native

# chain half — register `pub mod runtime_fiat_batch_acceptance;` in runtime/tests/transactions/mod.rs
cd quantus-chain
cargo test -p quantus-runtime --test mod \
  privacy_hidden_batch_payload_is_chain_accepted_and_transfers_one_million_quan -- --nocapture
```

### Observed output

```
RENDERED_CONFIRMATION=... | Balances · transfer_allow_death | Destination | qzmM9K... | Amount | ----- | ... | Approve | Cancel
INDEPENDENT_DECODE=Multisig.approve(id=0, Utility.batch([Balances.transfer_allow_death(to=qzmM9K..., amount=1000000000000000000 raw = 1000000 QUAN)]))
MLDSA_SIGNATURE_VERIFIED=true
CHAIN_ACCEPTED=true; TRANSFERRED_RAW=1000000000000000000; TRANSFERRED_QUAN=1000000
```

The mobile ML-DSA payload hex equals the runtime-accepted payload hex: the bytes signed under a blank amount are exactly the bytes that move the funds.

**Fix check.** Changing only `DecodedCallView` to render the chain-denominated native amount makes the sheet show `Amount: -1,000,000 QUAN`, and the unchanged mobile test then fails its `-----` assertion; a sweep over `batch` / `batch_all` / `force_batch`, nested batch, existential-deposit, and million-QUAN values is exact under the fix.

**Full PoC files:** <https://gist.github.com/vendrell46/f28de277cdd4571e5edaf4207dfe6d73>
