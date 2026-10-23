# #42937 \[BC-Insight] Public Exposure of Validator Signer Private Key in Executor Struct

**Submitted on Mar 29th 2025 at 19:09:12 UTC by @savi0ur for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42937
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**

## Description

## Bug Description

The `Executor` struct contains a field `pub signer: ValidatorSigner`. The `ValidatorSigner` type from `aptos_types::validator_signer` is designed to hold and manage a validator's private key for signing transactions. By declaring this field as `pub`, it becomes accessible to any code that can reference an `Executor` instance, exposing the private key to unintended access.

Here’s the relevant code block:\
<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/executor/mod.rs#L25>

```rust
pub struct Executor {
    /// The executing type.
    pub block_executor: Arc<BlockExecutor<AptosVM>>,
    /// The signer of the executor's transactions.
    pub signer: ValidatorSigner, //@audit-issue this field need to be private, as its holding validators private key.
    transactions_in_flight: Arc<RwLock<GcCounter>>,
    pub(crate) config: Config,
    pub(crate) node_config: NodeConfig,
}
```

```rust
#[derive(Debug)]
#[cfg_attr(any(test, feature = "fuzzing"), derive(Clone, Serialize, Deserialize))]
pub struct ValidatorSigner {
	author: AccountAddress,
	private_key: bls12381::PrivateKey,
}
```

Note: `ValidatorSigner` implements `Debug` trait, which could print this structure in a log, which will then expose validator's private key.

## Impact

Exposing the `signer` field publicly allows any part of the codebase—or even external crates if the `Executor` is re-exported to access the `ValidatorSigner` and, by extension, the validator’s private key. This can lead to:

* An attacker or malicious code could use the private key to sign fraudulent transactions, compromising the integrity of the blockchain.
* The private key could be inadvertently logged, or extracted, leading to a complete loss of validator's security.
* Validators are critical to the consensus mechanism; compromising their keys could destabilize the network.

## Recommendation

In Rust, `pub` fields are accessible outside the module and such sensitive fields should be kept private and only accessible via associated method.

Such sensitive informations should not be logged / serialized in production.

## Proof of Concept

## Proof Of Concept

* Validator's instantiate their `Executor` instance which also hold their `signer` details.
* `signer` holds a struct which contains private key and derives `Debug` trait. It means this struct is printable.
* We did not found any instance in the code that's printing `signer` / whole executor struct, but its not recommended to have this field public as it holds sensitive information.
