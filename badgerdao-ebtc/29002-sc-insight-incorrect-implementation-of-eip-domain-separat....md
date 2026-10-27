# 29002 - \[SC - Insight] Incorrect implementation of EIP- domain separat...

Submitted on Mar 4th 2024 at 13:58:49 UTC by @cheatcode for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #29002

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/BorrowerOperations.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

See below

## Vulnerability Details

EIP-712 provides a standard for signing typed structured data in Ethereum smart contracts, using the concept of domain separation to prevent collisions and replay attacks.

A key component of validation is the domain separator, which uniquely identifies the specific contract and chain for message signing purposes. The domain separator is constructed from:

* **DOMAIN\_TYPEHASH**: The keccak256 hash of the EIP-712 domain schema definition itself (the actual string types used like name, version etc.)
* **Contract Specific Values**: Domain values like name, version, chain ID and contract address.

In the current contract, the `_TYPE_HASH` variable used in the domain separator construction does not match the expected `DOMAIN_TYPEHASH`. Upon closer inspection, it is aligned closer to the hash of the full message type itself rather than the domain type schema.

## Impact Details

While this does not directly introduce security vulnerabilities per se, it indicates a gap in properly differentiating between DOMAIN\_TYPEHASH and actual message TYPEHASH as EIP-712 expects

## References

Add any relevant links to documentation or code

## Proof of Concept

This could lead to the following issues:

* Domain separators constructed incorrectly as per standards
* Signatures validated against such separators fail externally
* Mismatched assumptions between contract and clients
* Confusing for future audit and maintenance

**Recommendations**

1. Introduce a properly defined DOMAIN\_TYPEHASH constant
2. Construct domain separator by passing DOMAIN\_TYPEHASH and contract-specific values
3. Optionally validate both DOMAIN\_TYPEHASH and message TYPEHASH on chain

This will greatly improve standards compliance, clarity and future security.
