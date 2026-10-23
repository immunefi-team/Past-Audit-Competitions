# #41027 \[BC-Insight] Breaking asset surjection proof assumptions

**Submitted on Mar 9th 2025 at 17:58:59 UTC by @Blockian for** [**IOP | Zano**](https://immunefi.com/audit-competition/iop-zano)

* **Report ID:** #41027
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/zano-iop/tree/main/src/currency\\_core/currency\\_format\\_utils.cpp>
* **Impacts:**
  * Cryptographic inconsistency

## Description

Note:\
I'm relying on two checks in you code base:

* `static const size_t N_max = 256;` (src/crypto/one\_out\_of\_many\_proofs.cpp:23)
* `static_assert(CURRENCY_TX_MAX_ALLOWED_INPUTS <= N_max, "CURRENCY_TX_MAX_ALLOWED_INPUTS is inconsistent with one-out-of-many proof limits");` (src/crypto/one\_out\_of\_many\_proofs.cpp:26)
* `CHECK_AND_FAIL_WITH_ERROR_IF_FALSE(N <= N_max, 3);` (src/crypto/one\_out\_of\_many\_proofs.cpp:79)\
  They state that the aggregated surjection proof can only work up to 256 inputs.

**I don't know if by having more inputs than that the proof can be bypassed, or if it only breaks it so that it won't work at all**

## Impact

The impact is that `one out of many` `verify_BGE_proof` (used for aggregated asset surjection) can be called with a ring size that is larger than `256` which is the maximum that is consistent with the one-out-of-many proof (as per this comment: `src/crypto/one_out_of_many_proofs.cpp:23`)

## Root Cause

The root cause is that the number of inputs isn't validated anywhere.

## Deep Dive

* Can search `vin.size()` and `CURRENCY_TX_MAX_ALLOWED_INPUTS` accross the codebase to validate

## Suggested Fix

Validate `tx.vin.size()`

## Severity

As stated, I don't know.\
If this does nothing to the proof and the comment is wrong, than it's only an insight.\
If this breaks the proof, than it's dependent on what breaks. A critical if the checks can be completely bypassed in some instances, less otherwise.\
I'm not sure I have enough time to dive into this aggregation proof as I'm trying to cover as much code as possible.

## Proof of Concept

## Proof of Concept

Any transaction with more than 256 inputs.
