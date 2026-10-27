# 90235 bc high wormhole fee credits skip record transfer miner and aggregator volume fees are minted to keyless addresses with no zk tree leaf permanently freezing them

**Submitted on Aug 23rd 2026 at 22:30:18 UTC by @emp\_mrpl for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90235
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)
  * Permanent freezing of funds

## Description

## Summary

On every wormhole exit, `settle_batch` credits three parties. The exit recipient's credit is paired with `record_transfer`, which inserts a leaf into the zk-tree. The miner's volume-fee share and the aggregator's rebate are not — they are minted with `increase_balance` alone.

The withdrawal circuit can only ever spend value that is committed as a leaf under `ZkTree::Root`. A balance with no leaf therefore has zero exit capacity. Because both fee recipients are **keyless** addresses (derived by Poseidon2 re-hash, no signing key exists), the credited value can be moved neither by signature nor by exit. It is permanently frozen.

## Affected code (pinned commit `3b243b870a5442a6e5f443056f67bbba87d472f5`)

| Credit            | Site                                       | `record_transfer`? | Leaf?  | Spendable? |
| ----------------- | ------------------------------------------ | ------------------ | ------ | ---------- |
| Exit recipient    | `pallets/wormhole/src/lib.rs:854` → `:880` | yes                | yes    | yes        |
| Miner volume fee  | `pallets/wormhole/src/lib.rs:981`          | **no**             | **no** | **never**  |
| Aggregator rebate | `pallets/wormhole/src/lib.rs:945`          | **no**             | **no** | **never**  |

The miner address comes from `qp_wormhole::derive_wormhole_address` (`primitives/wormhole/src/lib.rs:109`), a Poseidon2 re-hash of the PoW pre-runtime digest. No private key produces it.

## Impact

* Fires on **100% of exits carrying a non-zero fee**, deterministically, on every honest node. No attacker, no crafted input, no privileged actor.
* Private exit: 5,000,000,000 planck frozen. Public exit: 7,500,000,000 planck (miner 5e9 + aggregator 2.5e9).
* The victims are the block author and the aggregator, who never receive a spendable form of the fee they earned.
* No recovery path. A forward fix cannot retroactively insert leaves for fees already credited in historical blocks; those credits remain uncommitted under every historical `ZkTree::Root`.

## Why this is "Permanent freezing of funds" and not "unclaimed yield"

Unclaimed yield is income not yet distributed. Here the distribution demonstrably ran: the runtime executes `increase_balance(&author, miner_fee, Exact)` at `:981`, emits `Event::MinerVolumeFeePaid` at `:987`, and the author's free balance rises from 0 to 5,000,000,000 (measured, both in a source-exact test and in live committed state at block 22). The value is already funds on a balance. A balance that can never be spent maps to "Permanent freezing of funds", and since the fix requires a runtime upgrade that does not restore historical credits, also to "Permanent freezing of funds (fix requires hardfork)".

## Not a duplicate of prior audit findings

**Eiger EQ-QNT-WORMHOLE-F-03** ("Miner Rewards Silently Burned Due to Incorrect Preimage Handling") is the direct ancestor. Pre-fix, the fee was minted with no `record_transfer` *and* the preimage was decoded directly, so a decode failure burned the fee. Eiger's remediation — deriving the address via Poseidon2 — made the fee land at a real keyless address, still with no leaf. **The F-03 fix converted a silent burn into a silent freeze.** This is a fix-introduced defect, not a duplicate: F-03 was wrong-address-to-burn; this is correct-address-but-no-leaf-to-freeze.

**EQ-QNT-WORMHOLE-R-08** shows that at audit time (2026-03-20) `record_transfer` did not insert a zk-tree leaf at all. The leaf mechanism and the exit's dependence on it are a post-audit change that never wired the fee credit sites.

## Not covered by the program's known issues

PROGRAM known issue #3 waives mining-reward emission dropping below the quantized minimum. This finding is on a different pallet and call path (`settle_batch` volume-fee distribution, not `pallet-mining-rewards` emission), and its stated mitigation (a tree-depth runtime upgrade) does not touch `:945`/`:981`.

The same keyless address demonstrates the contrast directly: its mining rewards DO get committing leaves (29 of them, verified by direct oracle query), while its volume fee gets none.

## Investigated and explicitly NOT claimed

Per-exit, `Σ(free balances)` rises while `TotalIssuance` stays flat, which looked like a candidate unauthorized-mint. It is not, and I am not claiming it:

1. The deposit locks the principal on a keyless vault (measured live: `0 → 2e13`), so the exit mint is a replacement of value already removed from circulation — textbook EIP-7503 lock-and-mint.
2. The exit circuit binds the exited leaf's recipient to `H(H("wormhole" ‖ secret))` (`unspendable_account.rs` `CircuitFragment::circuit`; `circuit.rs` `connect_shared_targets`). A leaf whose recipient is a signature-spendable account is not exitable without a Poseidon2 preimage break.

Both are documented in full in sections 10 and 11 of 01\_full\_report.md in the linked gist.

## Environment

All testing on a local `--dev` QPoW node only. No mainnet or public testnet was contacted, per program rules.

## Link to Proof of Concept

<https://gist.github.com/IbrahimMrpl/a6a0ee14d3e9923ec14bba1dceb0a412>

## Proof of Concept

The exploit path is entirely shipped code — no shim, no rebuild, no feature flag.

## Part A — live accepted exit on a real permissionless deposit

Node: local `quantus-node --dev`, port 9945, specVersion 143, pinned commit `3b243b870a5442a6e5f443056f67bbba87d472f5`.

{% stepper %}
{% step %}

## Deposit (permissionless, shipped path)

Pick a secret `s`, derive `U = UnspendableAccount::from_secret(s)`.

Submit a plain signed `Balances::transfer_allow_death(U, 20_000_000_000_000)` from a funded Dilithium account.

* Deposit tx: `0xa400cc6c…d762f95`
* The shipped runtime tx-extension `WormholeProofRecorderExtension` (`runtime/src/transaction_extensions.rs:339` Transfer match → `:389` → `pallet-wormhole record_transfer_proof :1367` → `record_transfer :1393`) records committed **leaf #35**, `transfer_count = 0`, `leaf_hash = 0x2c845015…5bb1`.
* The transfer itself locks the principal: `free(U)` goes `0 → 20,000,000,000,000`.

There is no dedicated deposit extrinsic; a plain signed transfer to an unspendable address IS the deposit mechanism.
{% endstep %}

{% step %}

## Generate a proof bound to live chain state

Read live header N = 16 (`parentHash`, `stateRoot`, `extrinsicsRoot`, `zkTreeRoot`, `digest`) and the real Merkle siblings/positions of leaf #35 (depth 3).

Run `qproof --mode gen-private` with `QP_LIVE=1`.

* The circuit's computed `block_hash = 0xd3184e30…8933`
* `chain_getBlockHash(16)` returns the identical value → the proof is provably bound to committed chain state, not a synthetic block.
  {% endstep %}

{% step %}

## Submit the exit

Submit the unsigned `verify_private_batch` (call\_index 2).

* **ACCEPTED.** Exit tx `0x33d5f9b3723887d9b409efd4bbbe2f9397af121b930bcd1c9bec1f29fca95eca`
* Included in **block 22**.
  {% endstep %}

{% step %}

## Read committed state (blocks 21 → 22)

Decoded wormhole events of block 22:

| Event                    | Recipient                            | Amount (planck)                 | zk-tree leaf?         |
| ------------------------ | ------------------------------------ | ------------------------------- | --------------------- |
| `NativeTransferred`      | exit recipient `0x0404…0404`         | 19,980,000,000,000              | **yes**               |
| `ProofVerified`          | —                                    | exit\_amount 19,980,000,000,000 | —                     |
| **`MinerVolumeFeePaid`** | keyless author `0x8b3c…272c`         | **5,000,000,000**               | **NO**                |
| `NativeTransferred`      | author `0x8b3c…272c` (mining reward) | 677,940,125,224                 | yes (separate pallet) |

Committed storage (hashed into the state root):

* `ZkTree::LeafCount`: **49 → 52 (+3)** = 1 exit recipient (`record_transfer`) + 2 mining-reward leaves. The keyless miner-fee credit added **0** leaves.
* `ZkTree::Root`: `0x5423be46…fef1` → `0xa10fe444…acf87`
* Author `0x8b3c…272c` free balance: **14,249,714,740,921 → 14,932,654,866,145** (`+682,940,125,224`), which decomposes exactly as `5,000,000,000` (keyless volume fee, no leaf) `+ 677,940,125,224` (mining reward, which *does* get its own leaf).
  {% endstep %}

{% step %}

## Negative controls (the node's own validator)

* 1-bit mutated proof → **REJECTED**.
* Replay of the accepted proof → **REJECTED** (nullifier now used on-chain).
  {% endstep %}
  {% endstepper %}

## Part B — the spendability oracle (direct query, not inference)

Oracle = `ZkTreeApi::get_merkle_proof` (`runtime/src/apis.rs:184`), folded through the vendor Poseidon2 `pallet_zk_tree::tree::hash_node`. This is Quantus's own code, not written for this PoC.

Scan the live tree for the keyless author `0x8b3c…272c`:

* **29 leaves** commit to that address — all mining rewards (amounts \~677,9xx,xxx,xxx, each distinct).
* **0 leaves** commit the `5,000,000,000` volume-fee value.
* Root spot-check: `proof.root == committed ZkTree::Root`, **PASS**.
* Negative control: 1-bit flip of a true leaf hash → recomputed root differs → **REJECTED**.

Same address, same block. One credit type gets a committing leaf, the other does not. No `get_merkle_proof` can ever return a path to the fee value, so no spend proof for it can be constructed.

## Part C — source-exact differential (one-command reproduction)

```bash
cargo test -p pallet-wormhole --lib -- --nocapture audit_v2
```

**Miner half** — driving the committed fixture bundle through the real `verify_private_batch`:

```
AUDIT_EVIDENCE leaves_before=0 leaves_after=1 num_exits=1 miner_fee=5000000000
  author_bal_before=0 author_bal_after=5000000000
```

**Aggregator half** — through the real `verify_public_batch → settle_batch`:

```
AUDIT_EVIDENCE_AGG leaves_before=0 leaves_after=1 num_exits=1 aggregator_rebate=2500000000
  agg_bal_before=0 agg_bal_after=2500000000 exit_recipients_got_leaves=1
```

In both arms the leaf delta equals the exit-recipient count only. The single difference between the exit credit and the fee credits is the `record_transfer` call.

**Negative control on the same machinery:** the settlement path's `storage_tail` prices and records the same insert correctly, so the divergence is a property of the two flagged sites, not of the measurement.

## Fix

Pair `record_transfer` with the fee credits at `pallets/wormhole/src/lib.rs:945` (aggregator rebate) and `:981` (miner volume fee), as is already done for the exit credit at `:854` → `:880`.

Note that this does not recover fees already credited in historical blocks — those remain uncommitted under every historical `ZkTree::Root`.
