# #41985 \[BC-Insight] Using the test keyring backend is insecure

**Submitted on Mar 19th 2025 at 19:34:52 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41985
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**

## Description

## Brief/Intro

The "test" keyring backend stores keys in plaintext which means that it is insecure.

## Vulnerability Details

Using a "test" keyring backend is not safe for production as it stores keys in plaintext. When handling real private keys, it is recommended to use a more secure keyring backend like "file" or "os".

Hence, a different keyring backend should be used for these two instances:

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/celestia-runners/src/celestia\\_light/mainnet.rs#L21>

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/setup/src/common/celestia.rs#L77>

## Impact Details

The "test" keyring backend stores keys in plaintext which is not secure.

## References

References are provided throughout the report.

## Proof of Concept

## Proof of Concept

According to the Celestia docs, there are multiple keyring backends that could be used instead: <https://docs.celestia.org/how-to-guides/celestia-app-vesting#optional-change-your-client-toml>
