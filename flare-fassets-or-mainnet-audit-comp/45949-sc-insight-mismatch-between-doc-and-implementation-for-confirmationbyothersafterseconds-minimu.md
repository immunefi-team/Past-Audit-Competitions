# #45949 \[SC-Insight] Mismatch between doc and implementation for \`confirmationByOthersAfterSeconds\` minimum on XRP

**Submitted on May 22nd 2025 at 20:36:08 UTC by @Rhaydden for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45949
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/SettingsInitializer.sol>
* **Impacts:**

## Description

## Issue description

The [doc](https://dev.flare.network/fassets/operational-parameters) specifies that the minimum value for `confirmationByOthersAfter` ("confirmationByOthersAfterSeconds") for XRP should be 6 hours (21,600 seconds). But, the songbird network config for f-xrp.json currently sets this value to 2 hours (7,200 seconds), and the code in `SettingsInitializer.sol` only enforces a minimum of 2 hours.

The parameters for BTC and DOGE are set correctly.

Relevant code/config snippets:

From songbird/f-xrp.json: <https://github.com/flare-labs-ltd/fassets//blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/deployment/config/songbird/f-xrp.json#L50>

```
"confirmationByOthersAfterSeconds": 7200,
```

From contracts/assetManager/library/SettingsInitializer.sol:

<https://github.com/flare-labs-ltd/fassets//blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/SettingsInitializer.sol#L81>

```
require(_settings.confirmationByOthersAfterSeconds >= 2 hours, "must be at least two hours");
```

(This line only enforces a 2-hour minimum for all assets.)

## Impact

The config and contract allow a value lower than the protocol-mandated minimum causing a mismatch between the actual behaviour and what is actually documented.

## Fix

Either update the doc or update the params in the contracts.

## Proof of Concept

## Proof of Concept
