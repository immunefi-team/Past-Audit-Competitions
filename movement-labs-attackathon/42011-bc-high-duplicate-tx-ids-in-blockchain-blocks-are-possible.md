# #42011 \[BC-High] Duplicate tx IDs in blockchain blocks are possible

**Submitted on Mar 20th 2025 at 01:21:02 UTC by @Rhaydden for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42011
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/util/movement-types>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

`Movement-types`has an issue where transactions with identical IDs but different application priorities can coexist in a block. This breaks the usual assumption of transaction ID uniqueness in blockchain prootocols.\
This is because transaction ID is calculated without considering the `application_priority`, enabling transactions with identical data and `sequence_number` but varying priorities to coexist. In protduction environs, this coulld cause double spending or unauthorized state changes.

## Vulnerability Details

This comes from a mismatch between how transaction IDs are computed and how transactions are ordered and deduplicated in blocks.

In `Transaction::new` method, transaction IDs are computed using only the transaction data and sequence number, excluding the application priority:

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/movement-types/src/transaction.rs#L52-L59>

```rust
impl Transaction {
	pub fn new(data: Vec<u8>, application_priority: u64, sequence_number: u64) -> Self {
		let mut hasher = blake3::Hasher::new();
		hasher.update(&data);
		hasher.update(&sequence_number.to_le_bytes());
		let id = Id(hasher.finalize().into());
		Self { data, sequence_number, application_priority, id }
	}
```

Albeit, the `Ord` implementation for `Transaction` prioritizes comparison by application priority first, before considering sequence number or ID:

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/movement-types/src/transaction.rs#L85-L102>

```rust
impl Ord for Transaction {
	fn cmp(&self, other: &Self) -> Ordering {
		// First, compare by application_priority
		match self.application_priority.cmp(&other.application_priority) {
			Ordering::Equal => {}
			non_equal => return non_equal,
		}

		// Then compare by sequence number
		match self.sequence_number().cmp(&other.sequence_number()) {
			Ordering::Equal => {}
			non_equal => return non_equal,
		}

		// If sequence number is equal, then compare by transaction on the whole
		self.id().cmp(&other.id())
	}
}
```

Now, blocks use a `BTreeSet<Transaction>` to store transactions, which relies on this `Ord` implementation to determine uniqueness:

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/movement-types/src/block.rs#L59-L64>

```rust
pub struct Block {
	metadata: BlockMetadata,
	parent: Id,
	transactions: BTreeSet<Transaction>,
	id: Id,
}
```

`Block::add_transaction` method simply inserts transactions into this set without checking for ID uniqueness:

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/movement-types/src/block.rs#L106-L108>

```rust
	pub fn add_transaction(&mut self, transaction: Transaction) {
		self.transactions.insert(transaction);
	}
```

This combination allows two transactions with identical data and sequence numbers (thus identical IDs) but different application priorities to coexist in the same block. The comparison returns based on the application priority difference before ever reaching the ID comparison, allowing both transactions to be inserted into the `BTreeSet`.

Let's consider this case for clarity;

Looking at the Ord implementation for Transaction again,

```rust
impl Ord for Transaction {
    fn cmp(&self, other: &Self) -> Ordering {
        // First, compare by application_priority
        match self.application_priority.cmp(&other.application_priority) {
            Ordering::Equal => {}
            non_equal => return non_equal,
        }

        // Then compare by sequence number
        match self.sequence_number().cmp(&other.sequence_number()) {
            Ordering::Equal => {}
            non_equal => return non_equal,
        }

        // If sequence number is equal, then compare by transaction on the whole
        self.id().cmp(&other.id())
    }
}
```

1. The comparison first checks `application_priority`
2. If `application_priority` values are equal, it checks `sequence_number`
3. Only if both `application_priority` and `sequence_number` are equal does it compare by `id`

Now, let's consider a scenario:

* Two transactions with the same `data` and `sequence_number` will have the same `id` (since `id` is computed from these two fields)
* If they have different `application_priority` values, the comparison will return based on the `application_priority` comparison, without ever reaching the `id` comparison.

note: `id` comparison is only reached if both `application_priority` and `sequence_number` are equal.

For a concrete example:

```rust
// These have the same data and sequence_number, thus the same id
let tx1 = Transaction::new(vec![1, 2, 3], 0, 1); // application_priority = 0
let tx2 = Transaction::new(vec![1, 2, 3], 1, 1); // application_priority = 1
```

When comparing these:

1. `tx1.application_priority (0) < tx2.application_priority (1)`
2. The comparison returns `Ordering::Less` immediately
3. The `id` comparison is never reached

For a `BTreeSet<Transaction>`, this means:

* Transactions are considered unique based on the full `Ord` implementation
* Two transactions with different `application_priority` values will be considered distinct even if they have the same `id`
* The `BTreeSet` will allow both to be inserted, as the uniqueness check is based on the entire ordering, not just the `id`

The `BTreeSet` will not prevent transactions with identical IDs from coexisting if they have different `application_priority` values because the comparison returns before reaching the `id` comparison.

Meaning that transactions with identical `ID`s but different `application_priority` values can indeed coexist in a block leading to the same transaction being processed multiple times.

## Impact Details

If a transaction reps a value transfer, processing the same transaction ID multiple times could casuse double-spending where the same funds are spent multiple times. Blockchain systems rely on deterministic state transitions. Duplicate transaction IDs is a cause for concern which could lead to inconsistent state updates across nodes.

## References

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/movement-types/src/transaction.rs#L52-L59>

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/movement-types/src/transaction.rs#L85-L102>

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/movement-types/src/block.rs#L59-L64>

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/movement-types/src/block.rs#L106-L108>

## Proof of Concept

## Proof of Concept

1. Create two transactions with identical data and sequence numbers but different application priorities:

```rust
let tx1 = Transaction::new(vec![1, 2, 3], 0, 1); // application_priority = 0
let tx2 = Transaction::new(vec![1, 2, 3], 1, 1); // application_priority = 1
```

2. Verify that both transactions have the same ID:

```rust
assert_eq!(tx1.id(), tx2.id());
```

3. Add both transactions to a block:

```rust
let mut block = Block::new(BlockMetadata::BlockMetadata, Id::test(), BTreeSet::new());
block.add_transaction(tx1);
block.add_transaction(tx2);
```

4. Verify that both transactions exist in the block despite having the same ID:

```rust
let transactions: Vec<_> = block.transactions().collect();
assert_eq!(transactions.len(), 2);
```

5. When this block is processed, both transactions will be executed

## Fix

There are 2 ways to go about the fix:

Either modify the `Transaction::new` method to include the application priority in the hash calculation for the transaction ID

OR

Make the `Block::add_transaction` method to check for existing transactions with the same ID before insertion. If a transaction with the same ID already exists in the block, either reject the new transaction or replace the existing one based on application priority.
