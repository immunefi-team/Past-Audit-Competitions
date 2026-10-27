# 89802 w a critical missing chain identity validation in the quantus sdk signing path lets a malicious rpc endpoint obtain signatures valid on an unintended chain

**Submitted on Aug 21st 2026 at 09:39:20 UTC by @bytes for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89802
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions

## Description

### Brief/Intro

The Quantus SDK fetches several fields of the chain's identity (genesis hash, spec version, transaction version) from the RPC endpoint that will receive the transaction and signs them without validating against any sort of local chain identity information. The SDK includes a genesis hash whitelist enforced on the cold wallet path, but the mobile signing path never calls it. A malicious RPC endpoint can choose the chain a user's signature is valid on. Since one seed derives the same address on every Quantus network, the signature can move the user's funds on a network they never selected.

### Vulnerability Details

**The chain ID is taken from the endpoint, unvalidated.** At `substrate_service.dart:365-371`, `_getGenesisHash` will just return whatever the server returns without a comparison.

**The genesis hash is signed as-is.** At `substrate_service.dart:214-233`, the endpoint's fields go into a `SigningPayload` and then line 255 calls `sign` without inspecting the `genesisHash`. The same pattern repeats at 292-311.

**The whitelist check exists in the same package, but is not called.** At `quantus_payload_parser.dart:34-38`, `knownNetworks` defines the genesis hashes the wallet will sign for, and `parsePayload` enforces it at 140-143. The cold wallet calls this (`sign_transaction_screen.dart:48`), but `substrate_service.dart` has no reference to `knownNetworks`, `parsePayload`, or `specMatchesBundled`.

### Impact Details

The genesis hash, spec version, and transaction version are parameters that are signed and therefore bind a signature to one chain. The endpoint supplies them and the wallet signs without checking any of them. The app's endpoint list is hardcoded to a single network so the correct chain is known at build time, but nothing compares against it. The confirmation screen shows the amount and recipient but never the chain so the user has no way to notice.

Funds at risk: One seed produces the same address on every Quantus network, so any user who holds a balance on more than one network can have funds moved on the unintended one. Every transaction signed through `substrate_service.dart` is affected. Cold wallet users are not affected.

Precondition: Control of one configured RPC endpoint. All endpoints are HTTPS, so this requires endpoint compromise rather than network interception.

## References

Affected code cited inline above.

## Proof of Concept

The attacker controls one of the three hardcoded RPC endpoints (`quantus_sdk/lib/src/constants/app_constants.dart:15-19`). All but two requests are proxied to a genuine Planck node, so the wallet behaves normally in every other respect.

{% stepper %}
{% step %}

### The user initiates a transfer

The Quantus SDK calls `getExtrinsicPayload` (`substrate_service.dart:213`), which issues five concurrent requests (lines 214-224):

* `getRuntimeVersion`
* `_getGenesisHash`
* `_getBlockNumber`
* `_getBlockHash`
* `_getNextAccountNonceFromAddress`
  {% endstep %}

{% step %}

### The attacker substitutes the genesis hash

`_getGenesisHash` calls `chain_getBlockHash` with param `[0]` to establish chain identity. The attacker replies with Heisenberg's genesis hash (`0xa5aa...3b4f`) instead of Planck's (`0x4901...e72`).
{% endstep %}

{% step %}

### The attacker supplies matching runtime versions

The attacker answers `state_getRuntimeVersion` with Heisenberg's spec and transaction versions to match.
{% endstep %}

{% step %}

### The SDK accepts the attacker-controlled hash

`_getGenesisHash` strips off the `0x` prefix and returns the attacker's value (line 371) without comparison. The wallet ships with a whitelist of the genesis hashes it will sign for (`quantus_payload_parser.dart:34-38`) and a function, `parsePayload`, that rejects any payload whose genesis hash is absent from it (lines 140-143). Neither is called anywhere in `substrate_service.dart`.
{% endstep %}

{% step %}

### The SDK builds the signing payload

The attacker's genesis hash, spec version, and transaction version are placed directly into `SigningPayload` with no inspection (line 233).
{% endstep %}

{% step %}

### The wallet signs for Heisenberg

Line 255 signs the payload with the user's ML-DSA key. Nothing between lines 233 and 255 inspects the genesis hash or any other part of the payload, and the resulting signature binds the transaction to Heisenberg.
{% endstep %}

{% step %}

### The user approves without seeing the chain

The confirmation screen shows the amount and recipient only. The chain the transfer will execute on is never displayed, so the user approves normally.
{% endstep %}

{% step %}

### The attacker submits the transaction to the unintended chain

The attacker withholds the extrinsic from Planck and submits it to Heisenberg instead, which accepts it because the signature commits to Heisenberg's genesis hash and runtime versions.
{% endstep %}

{% step %}

### The user's unintended-chain account is debited

The same seed derives the same address on every Quantus network, so the user holds that address on Heisenberg. That account is debited and the Planck transfer never occurs.
{% endstep %}
{% endstepper %}

The same sequence of events applies to the structurally identical second signing path at lines 292-311.
