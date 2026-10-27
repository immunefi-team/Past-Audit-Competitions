# 75576 bc medium proof executor mishandles destroyedchanged accounts halting proof generation

**Submitted on Apr 29th 2026 at 21:35:12 UTC by @Blobism for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75576
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

`TrieDB::update_accounts` in `crates/proof/executor/src/db/mod.rs` collapses all three revm "destroyed" account statuses (`Destroyed`, `DestroyedChanged`, `DestroyedAgain`) into a single "delete from trie and continue" branch. The `DestroyedChanged` case actually represents an account that was destroyed *and then re-created* in the same block, and must remain in the trie with new info and a wiped storage trie. Because of this mishandling, any block that contains the trivially reproducible pattern *deploy-via-CREATE2 → SELFDESTRUCT → re-deploy at the same address* causes the proof executor to abort with `TrieNodeError::KeyNotFound` and produces no state root. Since both the TEE and ZK proof-generation paths in production go through this code, no proof can be generated for the affected block and finalization on L1 stalls. Any unprivileged user can include such a transaction at any time, wedging the entire proof system and preventing the chain from finalizing new blocks.

## Vulnerability Details

`TrieDB::update_accounts` (`crates/proof/executor/src/db/mod.rs`, lines 164–228) is responsible for applying every `BundleAccount` in a block's bundle state to the in-memory MPT before recomputing the post-state root. The relevant excerpt is:

```rust
// crates/proof/executor/src/db/mod.rs:171-184
for (address, hashed_address, bundle_account) in sorted_state {
    if bundle_account.status.is_not_modified() {
        continue;
    }

    // Compute the path to the account in the trie.
    let account_path = Nibbles::unpack(hashed_address.as_slice());

    // If the account was destroyed, delete it from the trie.
    if bundle_account.was_destroyed() {
        self.root_node.delete(&account_path, &self.fetcher)?;
        self.storage_roots.remove(address);
        continue;
    }
    // ...account info + storage update + insert path follows...
}
```

`BundleAccount::was_destroyed()` (revm-database 10.0.0, `src/states/account_status.rs`) returns `true` for **three** distinct statuses:

```rust
pub fn was_destroyed(&self) -> bool {
    matches!(
        self,
        Self::Destroyed | Self::DestroyedChanged | Self::DestroyedAgain
    )
}
```

The semantic difference matters:

* `Destroyed` / `DestroyedAgain` — the account no longer exists at end of block, `info` is `None`. Deleting it from the trie is correct.
* `DestroyedChanged` — the account was destroyed *and then re-created* within the same block. `info` is `Some(new_info)`, `storage` contains the freshly created account's slots, and the account must be present at end of block with a freshly initialized storage trie.

reth's canonical state-root computation handles the `DestroyedChanged` case by wiping the existing storage trie *and* reinserting the new account info with the new storage:

```rust
// reth crates/trie/common/src/hashed_state.rs (HashedPostState::from_bundle_state)
let hashed_account = account.info.as_ref().map(Into::into);          // Some(new_info)
let hashed_storage = HashedStorage::from_plain_storage(
    account.status,                                                  // wiped = was_destroyed()
    account.storage.iter().map(|(slot, value)| (slot, &value.present_value)),
);
```

The proof executor's `update_accounts` does neither: it only deletes the account, never re-inserts it. Worse, in the typical scenario that produces a `DestroyedChanged` status the account did not exist at block start at all (it is a brand-new contract address that was created *inside this block*). Calling `TrieNode::delete` on a path that has no leaf in the parent trie returns `TrieNodeError::KeyNotFound` (see `crates/proof/mpt/src/node.rs:316–353`, `Self::Empty => Err(TrieNodeError::KeyNotFound)` and the `Leaf` arm).

`update_accounts` propagates this error as `TrieDBError`, `state_root` propagates it again, and the stateless block executor aborts the block. The proof executor produces no post-state root for the block.

### How an attacker triggers `DestroyedChanged`

EIP-6780 (active on Base) restricts `SELFDESTRUCT` so that it only deletes an account when that account was created in the *same transaction*. A previously-existing account therefore cannot reach `Destroyed` status, but a freshly deployed account can:

{% stepper %}
{% step %}

## Deploy a factory contract

The attacker deploys (or already has access to) a small factory contract. No special privileges are required.
{% endstep %}

{% step %}

## Create and selfdestruct a fresh contract

**Tx 1 in the target block:** the factory uses `CREATE2` with salt `s` and init-code `c` to deploy contract `A` at address `addr = CREATE2(factory, s, c)`. The constructor of `A` immediately invokes `SELFDESTRUCT`. Because `A` was created in this same transaction, EIP-6780 permits the deletion; revm marks `A`'s cache-state status as `Destroyed`, and `addr` is empty at end of tx 1.
{% endstep %}

{% step %}

## Re-deploy to the same address

**Tx 2 in the same block:** the factory invokes `CREATE2` again with the *same* salt `s` and the same init-code `c`. CREATE2 collisions are deterministic — the address resolves to the same `addr`. `addr` is empty post-tx-1, so the deployment succeeds. revm transitions `Destroyed.on_created() → DestroyedChanged`.

(The same scenario can also be staged inside a single transaction, e.g. by performing two `CREATE2`s with the same salt around an interleaved `SELFDESTRUCT` of the first deployment.)
{% endstep %}
{% endstepper %}

When the block finishes, the bundle merge logic for the `DestroyedChanged` arm (`BundleAccount::update_and_create_revert`, revm-database `bundle_account.rs:255–326`) sets:

```rust
self.status = AccountStatus::DestroyedChanged;
self.info   = updated_info;            // Some(new_info)
extend_storage(&mut self.storage, updated_storage);
```

So `bundle.state()` yields one entry for `addr` with status `DestroyedChanged` and a non-`None` `info`.

When `update_accounts` reaches this entry:

* `bundle_account.status.is_not_modified()` — `false`.
* `bundle_account.was_destroyed()` — `true`.
* `self.root_node.delete(&account_path, &self.fetcher)` is called against the parent state trie. Because `addr` did not exist at block start, the trie has no leaf for it, and `TrieNode::delete` returns `TrieNodeError::KeyNotFound`.

The error short-circuits `update_accounts` → `state_root` → the stateless block executor, and no post-state root is produced.

## Impact Details

For every block that contains the create-destroy-recreate pattern described above:

* `base-reth-node` (the canonical L2 execution layer) executes the block normally and produces a valid post-state root that reflects the recreated account.
* The proof executor in `crates/proof/executor` cannot complete `state_root(...)` and returns an error.

Both production proof-generation paths route through `TrieDB::state_root`:

* The TEE proposer/challenger (`base-tee-*`).
* The ZK prover (`base-succinct/*`).

Therefore *no* proof — TEE attestation or ZK proof — can be generated for the affected block. Per the Base Azul dispute game, finalization of an L2 batch on L1 requires a TEE or ZK proof. The affected block, and every block built on top of it, is unfinalizable until the bug is fixed and a fixed proof executor is redeployed.

The trigger requires:

* No privileged keys or roles.
* No special protocol state.
* A single user-level transaction containing two `CREATE2` calls and one `SELFDESTRUCT`, paid for with ordinary gas.

An adversary can submit one such transaction per block indefinitely, halting finalization for the entire network at trivial cost. This is a complete-network-shutdown class issue: the chain keeps producing unsafe blocks, but none of them can ever be proven and finalized to L1.

The bug affects blocks proven by either the TEE or the ZK pipeline because both share the same `crates/proof/executor` code. There is no fallback proof system that bypasses the affected code path.

## References

Commit: `e3467a2048881213b56739a54a876efb9c6ea103` (`v0.8.0-rc.28`)

* Vulnerable code: `crates/proof/executor/src/db/mod.rs:164-228` (`TrieDB::update_accounts`), specifically the unconditional `was_destroyed()` branch at lines 179–184.
* `TrieNode::delete` returning `KeyNotFound` for non-existent paths: `crates/proof/mpt/src/node.rs:316-353`.
* revm `was_destroyed()` definition (covers all three destroyed statuses): `revm-database-10.0.0/src/states/account_status.rs`.
* revm `BundleAccount::update_and_create_revert`, `DestroyedChanged` arm: `revm-database-10.0.0/src/states/bundle_account.rs:255-326`.
* reth canonical handling of `DestroyedChanged` (wipe storage trie + reinsert account): `reth crates/trie/common/src/hashed_state.rs`, `HashedPostState::from_bundle_state`.
* EIP-6780 (`SELFDESTRUCT` restricted to same-transaction creations).
* Base Azul dispute game / proof-required finalization flow (TEE and ZK pipelines both rely on `crates/proof/executor`).

## Link to Proof of Concept

<https://gist.github.com/blobism/256477d9eab8739ded13c2a6c8578458>

## Proof of Concept

Get the Gist from: <https://gist.github.com/blobism/256477d9eab8739ded13c2a6c8578458>

```bash
git apply poc.diff
```

### Build & run

From the repository root:

```bash
# Build the crate's tests (compiles base-proof-executor with std for tests).
cargo build -p base-proof-executor --tests

# Run the two unit tests.
cargo test -p base-proof-executor --lib db::tests::destroyed_changed
```

Expected output:

```
running 2 tests
test db::tests::destroyed_changed_on_fresh_address_aborts_state_root ... ok
test db::tests::destroyed_changed_on_existing_address_drops_account_silently ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 6 filtered out
```

Both tests passing demonstrates the bug:

* `destroyed_changed_on_fresh_address_aborts_state_root` — `update_accounts` (and therefore `state_root`) returns `TrieNodeError::KeyNotFound` on a `BundleState` with a single `DestroyedChanged` entry at a fresh address against an empty parent trie. This is the proof-generation-halting failure mode.
* `destroyed_changed_on_existing_address_drops_account_silently` — when the address *did* exist in the parent trie, `update_accounts` silently drops it (`get_trie_account` returns `None` afterwards) instead of preserving the account with the new info, diverging from reth's canonical `HashedPostState::from_bundle_state` representation.
