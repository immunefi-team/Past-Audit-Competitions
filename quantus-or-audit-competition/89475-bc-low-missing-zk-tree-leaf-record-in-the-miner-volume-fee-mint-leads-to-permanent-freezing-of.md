# 89475 bc low missing zk tree leaf record in the miner volume fee mint leads to permanent freezing of the miner s fee share

**Submitted on Aug 19th 2026 at 14:38:28 UTC by @BenTimor for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89475
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

I found that the miner's share of the exit volume fee in the zk-wormhole pallet is minted to an address from which it can never be spent. When `process_exit_bundle` settles an exit, the miner (block author) is credited 50% of the 4-bps volume fee at their *wormhole* address — a keyless address derived as `H(H("wormhole" || preimage))` — but, unlike every other mint on the chain, this credit never records a zk-tree leaf. Spending from a wormhole address is only possible via a zero-knowledge exit that proves a merkle path from a recorded leaf to the tree root; without a leaf there is no path, and without a private key there is no signature path. The fee is therefore permanently frozen at the author's address (or burned outright if the mint itself fails), and the miner never receives the compensation the protocol explicitly promises them. This happens in every exit bundle, forever: 2 bps of all exit volume is continuously minted into a black hole. No user funds are at risk; the loss is the miners' promised compensation, which the protocol silently fails to deliver.

## Vulnerability Details

### The two address kinds and the one-credit-one-leaf invariant

Quantus has two kinds of addresses that are indistinguishable on-chain. *Dilithium* addresses are ordinary keyed accounts (`H(public_key)`): funds move by signature. *Wormhole* addresses are deliberately keyless, derived from a secret preimage (`primitives/wormhole/src/lib.rs`):

```rust
pub fn derive_wormhole_address(inner_digest: [u8; 32]) -> Result<[u8; 32], &'static str> {
	rehash_to_bytes(&inner_digest)   // H(H("wormhole" || secret)) — no key exists for this
}
```

Funds at a wormhole address can only move through a zero-knowledge exit: the prover demonstrates knowledge of the secret behind the address, and the exit circuit enforces that the balance being exited is backed by a **zk-tree leaf** — a leaf committed by `record_transfer` at credit time. The exit proof carries a merkle path from that leaf to the tree root. This is the chain's core accounting invariant: **one credit = one leaf**. A balance without a leaf has no merkle path, hence no possible exit proof, and no key to sign with. It is permanently unspendable.

### The fee split

On every exit, the pallet charges a volume fee (4 bps, `VolumeFeeRateBps`) and splits it (`pallets/wormhole/src/lib.rs`, `process_exit_bundle`, lines \~879–890):

```
// Fee split (controlled by VolumeFeesBurnRate):
//   - burn_amount = fee * burn_rate  (reduces total issuance via Currency::burn)
//   - miner_fee = fee - burn_amount  (minted to block author via increase_balance)
```

With the production burn rate of 50%, the miner's share is 2 bps of every exited amount.

### The three mints — two correct, one broken

The same function performs three mints. Two of them respect the invariant:

**1. Exit credits** (lines \~815–858) — `increase_balance` **followed by `record_transfer`**, with a comment explaining the double-record hazard and confirming the leaf is deliberate:

```rust
// NOTE: this must stay `Unbalanced::increase_balance` (event-free). The runtime's
// `WormholeProofRecorderExtension` records a transfer proof for every `Balances::Minted`
// event it scans, and this exit already records its own proof via `record_transfer` below
// — switching to `mint_into` (which emits `Minted`) could double-record the credit.
```

```rust
Self::record_transfer(T::AssetId::default(), &from_account, &to_account, *exit_balance);
```

**2. Aggregator rebate** (lines \~897–931) — `increase_balance` to the aggregator's address, no leaf. This is correct, because the aggregator supplies a *keyed* dilithium address: the rebate is spendable by signature, so no leaf is needed.

**3. Miner fee** (lines \~944–977) — this is where the invariant breaks:

```rust
if let Some(author) = qp_wormhole::extract_author_from_digest::<...>(digest.logs.iter()) {
    match <T::Currency as Unbalanced<_>>::increase_balance(
        &author,
        miner_fee,
        frame_support::traits::tokens::Precision::Exact,
    ) {
        Ok(_) => {
            Self::deposit_event(Event::MinerVolumeFeePaid { miner: author, amount: miner_fee });
        },
        ...
```

`extract_author_from_digest` (`primitives/wormhole/src/lib.rs:122`) returns the block author as a **wormhole address** (`derive_wormhole_address(preimage)` — keyless, by design of the mining flow). The branch credits it with `increase_balance` and **never calls `record_transfer`**. It is structurally identical to the aggregator-rebate branch — but the aggregator's address is keyed and the author's is not. That difference is invisible in the code and in the types (both are `AccountId`), and it is exactly the difference between "spendable" and "frozen forever".

### Why the fee can never move

I checked every possible path out of this address:

* **Signature spending:** impossible — no private key exists for `H(H("wormhole" || preimage))`; a wormhole address is keyless by construction.
* **ZK exit:** impossible — the exit circuit requires a merkle path from a recorded leaf to the tree root. Exits are leaf-bounded, not balance-bounded. No leaf was recorded for this credit, so no proof can ever be constructed. The miner knowing their own preimage does not help: knowledge of the secret alone produces no leaf.
* **Any sweep/re-key/rescue:** none exists. The block-1 sweep is block-1-only, `force_transfer` is governance-Root-gated (out of scope), and there is no aggregate-exit path that covers unleafed balances.
* **The failure path is no better:** if the mint itself fails (e.g. author account below the existential deposit), the fee is added to the burn amount (lines \~968–977); if no author is found, the fee is burned (lines \~978–984). The miner loses their share in **every** outcome.

### Why the existing test cannot catch this

The pinned test `test_verify_private_batch_emits_miner_volume_fee_paid` (`pallets/wormhole/src/tests.rs:1036`) asserts the event fired and `Balances::free_balance(author) == before + fee`. Both are true *with* the bug. It never checks `TransferCount` (the leaf counter) for the author, and it would still pass unchanged if the bug were fixed. The test pins "the balance went up", not "the money can move" — and the balance does go up, into a black hole.

## Impact Details

* **What is lost:** 50% of the 4-bps exit volume fee — i.e. 2 bps of every exited amount — is minted into an unspendable keyless address (or burned), in **every exit bundle, continuously and permanently**. The protocol's own accounting comments, the `MinerVolumeFeePaid` event, and the pinned test all confirm this money was intended as miner compensation.
* **Who loses:** every miner who authors a block containing exit transactions. The compensation promised by the protocol's fee design is silently never delivered.
* **What is NOT at risk:** no user funds are touched, no existing balance is frozen, nothing is stolen, and the spendable supply is unaffected (the frozen mint is unspendable from the moment of creation — it is the economic equivalent of a burn, without the burn half's deflationary accounting). Consensus and the ZK system are unaffected.
* **Secondary effect:** the miner incentive to include exit transactions is quietly removed, which works against the liveness/throughput of exits over time.

**Severity: Low.** This maps directly to the program's in-scope impact **"Contract fails to deliver promised returns, but doesn't lose value"**: the chain promises the miner a fee share and fails to deliver it, while no one loses value they already held. It is not the Critical impact "Permanent freezing of funds" — that entry covers user funds, whereas the frozen tokens here are freshly minted compensation that never belonged to anyone; and per the program page, "the governance system handles runtime upgrades, which can edit storage, which can undo most problems" (a runtime upgrade can add `record_transfer` to the fee branch), which further supports Low rather than a hardfork-class severity.

## Proof of Concept

*The program requires a runnable PoC. I provide a runnable Rust test that drives the real `verify_private_batch` path with the repository's real ZK fixture proof, plus the step-by-step explanation below. The full test source is attached as `poc_miner_fee_tests.rs`; it can be dropped into `pallets/wormhole/src/tests.rs` and run directly.*

{% stepper %}
{% step %}

## Reproduce the intended behavior

**Prerequisites:** clone the frozen competition code `immunefi-team/audit-comp-quantus-chain` (branch `audit-comp-ready`, commit `3b243b8`) and build the test suite (`cargo test -p pallet-wormhole`).

Run the repository's own pinned test:

```
cargo test -p pallet-wormhole --lib --release test_verify_private_batch_emits_miner_volume_fee_paid -- --nocapture
```

It passes: after `verify_private_batch` processes a real exit proof, the event `MinerVolumeFeePaid` fires and `Balances::free_balance(author)` increased by exactly the expected miner fee. The author here is `derive_wormhole_address([7u8; 32])` — a keyless wormhole address.
{% endstep %}

{% step %}

## Prove the leaf is missing

Add the attached test `poc_miner_volume_fee_mint_has_no_zk_leaf` (included in `poc_miner_fee_tests.rs`). It runs the same `verify_private_batch` call and then asserts, for the author's canonical address:

```rust
// 1. The fee IS minted to the author (existing pinned behavior).
assert!(Balances::free_balance(expected_author) > author_balance_before);

// 2. ...but NO zk-tree leaf is recorded for the author: TransferCount unchanged.
assert_eq!(author_count_after, author_count_before,
    "miner fee mint must not create a zk-tree leaf for the author");

// 3. The exit credits DID insert leaves (the invariant works for exits)...
assert!(leaves_after.len() > leaves_before.len());

// 4. ...and none of the new leaves credits the author.
for index in new_leaves {
    let leaf = pallet_zk_tree::Leaves::<Test>::get(index);
    assert_ne!(leaf.to, expected_author, "no new leaf may credit the miner fee to the author");
}
```

{% endstep %}

{% step %}

## Run the test

```
cargo test -p pallet-wormhole --lib --release poc_miner_volume_fee_mint_has_no_zk_leaf -- --nocapture
```

Result: **PASS** (69/69 tests in the suite green). The test proves simultaneously that (a) the fee balance is credited to the author, and (b) the author's `TransferCount` is unchanged and none of the newly inserted zk-tree leaves credits the author — the fee credit exists in the ledger but is not backed by any leaf.
{% endstep %}

{% step %}

## Consequence

The credited balance sits at `H(H("wormhole" || preimage))`. It cannot be signed for (no key exists for a wormhole address) and cannot be ZK-exited (the exit circuit requires a merkle path from a recorded leaf; exits are leaf-bounded and no leaf exists for this credit). Every additional exit bundle mints a further fee share into the same state. The funds are permanently frozen.
{% endstep %}
{% endstepper %}
