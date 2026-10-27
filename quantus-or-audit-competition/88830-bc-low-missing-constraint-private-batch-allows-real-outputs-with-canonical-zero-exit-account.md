# 88830 bc low missing constraint private batch allows real outputs with canonical zero exit account

**Submitted on Aug 15th 2026 at 05:18:28 UTC by @bugtester for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88830
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits>
* **Impacts:**
  * Modification of transaction fees outside of design parameters

## Description

## Brief/Intro

The private-batch aggregation circuit does not enforce that a real output with a non-zero amount must have a non-zero exit account.

The leaf-level circuit exposes the exit accounts as public inputs without constraining them to be non-zero. The private-batch circuit distinguishes dummy proofs using the block-hash sentinel and only masks exits to zero for those dummy proofs. For a real proof, the exit account is forwarded unchanged.

As a result, the circuit accepts the following state:

```
real proof
output_amount > 0
exit_account = [0, 0, 0, 0]
```

The private-batch aggregation logic then treats the zero exit account as a normal grouping key, aggregates its associated amount, and places the resulting `(amount, zero-exit)` pair into the aggregated public inputs.

This breaks the invariant that real positive-value exits must have a valid destination account.

## Vulnerability Details

<https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/blob/2a6914ed0184fbc0aaf621801ac8b8b7e3b01f6e/wormhole/aggregator/src/private_batch/circuit/circuit_logic.rs#L329-L376>

The private-batch circuit first determines whether a child proof is a dummy based on the block-hash sentinel.

For each output slot, dummy proofs are masked to:

```
exit = [0, 0, 0, 0]
amount = 0
```

For a real proof, `is_dummy_i` is false and the original `exit_raw` value is returned unchanged.

The subsequent aggregation logic compares exit accounts, groups matching exits, sums their amounts, and emits the resulting exit and amount.

There is no constraint enforcing:

```
real && amount > 0 => exit_account != 0
```

Therefore, the aggregation circuit does not prevent a real positive-value output from using the same canonical zero value that is otherwise used for dummy/unused exits.

## Impact Details

The missing constraint allows a real, positive-value exit to be represented with the canonical zero exit account.

The private-batch circuit uses the zero exit account as the canonical representation for dummy/unused outputs, but does not enforce that the same value cannot be used by a real output carrying a non-zero amount.

A real leaf can contain:

```
block_hash != 0
output_amount > 0
exit_account = 0
```

The private-batch logic does not reject this combination. Because real slots are forwarded without validating the exit account, the positive amount enters the aggregation logic under the zero exit key and is exposed in the aggregated public inputs as:

```
(amount > 0, exit_account = 0)
```

This breaks the expected invariant that every real positive-value exit has a valid destination account.

The ZK circuit therefore accepts and proves an invalid settlement state: a positive-value output is associated with the canonical zero destination.

If the downstream settlement layer processes the aggregated output without independently rejecting the zero exit account, the associated value can be sent to the zero/ownerless destination, resulting in permanent loss or misdirection of funds.

Even where downstream validation rejects the zero destination, the circuit still permits an invalid positive-value zero-exit state to become part of a valid aggregated proof, allowing malformed settlement batches to reach the settlement layer and potentially cause settlement failure or batch-level disruption.

## Proof of Concept

The issue can be reproduced by constructing a real leaf proof with a non-zero output amount and a zero exit account, then passing that proof through the private-batch aggregation circuit.

The relevant code paths are:

* **Private-batch masking + grouping:**

  <https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/blob/audit-comp-ready/wormhole/aggregator/src/private_batch/circuit/circuit_logic.rs#L329-L376>

  This shows that dummy slots are masked using `select(is_dummy_i, zero, exit_raw)` and `select(is_dummy_i, zero, amount_raw)`, while real slots are forwarded without a check enforcing `(real && amount > 0) => exit != zero`.

  The subsequent grouping/sum logic treats the exit account as a grouping key without validating that a positive amount has a non-zero destination.

  ```rust
  // Masked per-slot (exit, amount): dummy slots read as (zero account, 0).
  let mut slot_exits: Vec<[Target; 4]> = Vec::with_capacity(num_exit_slots);
  let mut slot_amounts: Vec<Target> = Vec::with_capacity(num_exit_slots);

  for slot in 0..num_exit_slots {
      let proof_idx = slot / 2;
      let (exit_raw, amount_raw) = get_exit_and_amount(proof_idx, slot % 2);
      let is_dummy_i = is_dummy_flags[proof_idx];

      slot_exits.push(core::array::from_fn(|j| {
          builder.select(is_dummy_i, zero, exit_raw[j])
      }));

      slot_amounts.push(builder.select(is_dummy_i, zero, amount_raw));
  }

  for slot in 0..num_exit_slots {
      let exit_slot = slot_exits[slot];

      // Check whether this exit appeared earlier (for dedupe)
      let mut is_duplicate = builder._false();

      for exit_earlier in slot_exits.iter().take(slot) {
          let matches_earlier = bytes_digest_eq(builder, *exit_earlier, exit_slot);
          is_duplicate = builder.or(is_duplicate, matches_earlier);
      }

      // Sum all matching amounts across all 2*N outputs
      let mut acc = zero;

      for (exit_j, amount_j) in slot_exits.iter().zip(&slot_amounts) {
          let matches = bytes_digest_eq(builder, *exit_j, exit_slot);
          let conditional_amount = builder.select(matches, *amount_j, zero);
          acc = builder.add(acc, conditional_amount);
      }

      // Zero duplicates so they look like dummy/unused slots
      let final_sum = builder.select(is_duplicate, zero, acc);

      let final_exit = [
          builder.select(is_duplicate, zero, exit_slot[0]),
          builder.select(is_duplicate, zero, exit_slot[1]),
          builder.select(is_duplicate, zero, exit_slot[2]),
          builder.select(is_duplicate, zero, exit_slot[3]),
      ];

      // Range check final sum to 32 bits
      builder.range_check(final_sum, 32);

      output_pis.push(final_sum);
      output_pis.extend_from_slice(&final_exit);
  }
  ```
* **SubstrateAccount leaf-level handling:**

  <https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/blob/audit-comp-ready/wormhole/circuit/src/substrate_account.rs#L147-L151>

  The exit accounts are exposed as public inputs without a constraint requiring them to be non-zero.
* **Formal specification:**

  <https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/blob/audit-comp-ready/formal/SPEC.md#L158-L201>

  The specification notes that `exit_account_1/2` are unconstrained at the leaf level and that the binding obligation is intended to be enforced at the private-batch layer.

## Recommended Fix

Add a circuit constraint requiring every real positive-value output to have a non-zero exit account:

```
is_dummy OR amount == 0 OR exit != zero
```

For example:

```rust
let exit_is_zero =
    bytes_digest_eq(builder, exit_raw, [zero, zero, zero, zero]);

let amount_is_zero =
    builder.is_equal(amount_raw, zero);

let exit_nonzero =
    builder.not(exit_is_zero);

let allowed =
    builder.or(is_dummy_i, builder.or(amount_is_zero, exit_nonzero));

builder.connect(allowed.target, one);
```

This ensures that:

```
real + amount > 0 + exit == zero
```

cannot satisfy the circuit.
