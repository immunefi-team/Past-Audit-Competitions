# #41669 \[BC-Medium] Incorrect Gas Cost Used for BLS12381 Subgroup Check Causes \~70% Undercharge

**Submitted on Mar 17th 2025 at 12:59:39 UTC by @Minato7namikazi for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41669
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Details

In attackathon-movement-aptos-core/move\_framework/src/natives/cryptography/bls12381.rs

In `bls12381_pk_subgroub_check`, the code charges `BLS12381_PER_PUBKEY_DESERIALIZE` instead of using `BLS12381_PER_PUBKEY_SUBGROUP_CHECK`. This causes the gas cost for subgroup-checking public keys to be incorrectly computed.

This is a clear issue since the gas cost should reflect the actual operation being performed - subgroup checking is a different operation from deserialization with different computational costs.

The function `bls12381_pk_subgroub_check` is specifically for checking subgroup membership

The gas parameters struct clearly defines separate costs for deserialization (per\_pubkey\_deserialize) and subgroup checking (per\_pubkey\_subgroup\_check)

## Impact

Looking at the gas schedule configuration in the Aptos framework, which defines the gas costs for various native operations in the blockchain.

* `BLS12381_PER_PUBKEY_DESERIALIZE` = 400,684 gas units
* `BLS12381_PER_PUBKEY_SUBGROUP_CHECK` = 1,360,120 gas units

The bug is charging 400,684 gas units when it should be charging 1,360,120 gas units\
This means users are being undercharged by 959,436 gas units (1,360,120 - 400,684) for each public key subgroup check operation

The gas cost being charged is only \~29.5% of what it should be

This represents a significant undercharging of approximately 70.5% less gas than what should be charged

Subgroup checking is a more computationally expensive operation than deserialization\
The gas schedule was designed to reflect this higher computational cost (hence the \~3.4x higher gas cost)

This could potentially be exploited by malicious users to perform expensive subgroup check operations while paying much less gas than they should

## Proof of Concept

1. Generate a BLS12381 public key that requires subgroup checking
2. If we Call bls12381\_pk\_subgroub\_check and measure the gas consumption it will show 400,684 units
3. In the gas schedule the BLS12381\_PER\_PUBKEY\_SUBGROUP\_CHECK should be 1,360,120 units
4. We Demonstrated the 959,436 gas unit deficit (70.5% undercharge) between actual vs expected cost
