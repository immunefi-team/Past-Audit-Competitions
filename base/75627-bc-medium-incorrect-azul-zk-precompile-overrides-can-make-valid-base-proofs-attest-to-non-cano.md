# 75627 bc medium incorrect azul zk precompile overrides can make valid base proofs attest to non canonical withdrawal state

**Submitted on Apr 30th 2026 at 06:06:20 UTC by @z41zen for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75627
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Direct loss to Base or users ≥ 10% of funds held within Bridge.
  * Unintended permanent chain split requiring hard fork (network partition requiring hard fork)

## Description

### Brief/Intro

Base Azul's ZK executor constructs the correct Azul precompile table and then overwrites parts of it with legacy accelerated precompile entries. As a result, the same L2 transaction can execute differently in the ZK executor than it does in canonical Base Azul EL.

This breaks the proof system's core safety property. The ZK executor can attest to a non-canonical output root, including a `L2ToL1MessagePasser` storage root containing a withdrawal that canonical Base EL never created. Once such an output root is accepted by the L1 bridge flow, the withdrawal can be proven and finalized against that root.

## Vulnerability Details

The vulnerable code is in `crates/proof/succinct/utils/client/src/precompiles/mod.rs`.

For `OpSpecId::AZUL`, the ZK precompile provider correctly starts from the canonical Base Azul precompile table:

```rust
OpSpecId::AZUL => BasePrecompiles::azul().clone(),
```

Immediately afterward, it extends that table with `get_precompiles()`:

```rust
let mut precompiles = base;
precompiles.extend(get_precompiles());
```

The problem is that `get_precompiles()` returns legacy accelerated entries:

```rust
fn get_precompiles() -> Vec<PrecompileWithAddress> {
    vec![
        bn254::add::ISTANBUL,
        bn254::mul::ISTANBUL,
        bn254::pair::ISTANBUL,
        secp256k1::ECRECOVER,
        secp256r1::P256VERIFY,
        kzg_point_evaluation::POINT_EVALUATION,
    ]
}
```

`Precompiles::extend` replaces entries at matching addresses. Therefore, the ZK executor first creates the correct Azul table and then overwrites Azul entries with older rules.

The clearest exploitable mismatch is `P256VERIFY` at `0x0100`:

* Canonical Base Azul uses `P256VERIFY_OSAKA`, which costs `6900` gas.
* The ZK executor overwrites that entry with legacy `P256VERIFY`, which costs `3450` gas.

Canonical Base Azul defines the correct table in `crates/common/evm/src/precompiles/provider.rs`:

```rust
pub fn azul() -> &'static Precompiles {
    static INSTANCE: OnceLock<Precompiles> = OnceLock::new();
    INSTANCE.get_or_init(|| {
        let mut precompiles = Self::jovian().clone();

        // Base Azul adopts Osaka pricing and bounds for MODEXP and P256VERIFY.
        precompiles.extend([modexp::OSAKA, secp256r1::P256VERIFY_OSAKA]);

        precompiles
    })
}
```

This incorrect provider is used by the actual ZK execution path. `ZkvmOpEvmFactory` constructs EVMs with `OpZkvmPrecompiles::new_with_spec(spec_id)`, and the witness executor passes `ZkvmOpEvmFactory::new()` into `BaseExecutor::new(...)`. Therefore, range proof execution uses the overwritten ZK precompile provider rather than the canonical Base EL provider.

This is externally observable from normal EVM bytecode. A contract can call `P256VERIFY` at `0x0100` with exactly `3450` gas and branch on the `CALL` success bit:

* Under canonical Base Azul EL, the call fails because `P256VERIFY_OSAKA` requires `6900` gas.
* Under the ZK executor, the same call succeeds because the overwritten legacy entry costs `3450` gas.

The `CALL` success bit is visible to user contracts. This means the same transaction can produce different storage writes, logs, and calls depending on whether it is executed by canonical Base EL or by the ZK executor.

The bridge-relevant PoC uses that divergence as a gate:

1. The contract calls `P256VERIFY` at `0x0100` with exactly `3450` gas.
2. If the call succeeds, it calls `L2ToL1MessagePasser` at `0x4200000000000000000000000000000000000016`.
3. Canonical Base EL does not reach the MessagePasser branch.
4. The ZK executor does reach the MessagePasser branch for the same bytecode.

The standalone PoC prints:

```
canonical_base_el_p256_call_success_word=0x0000000000000000000000000000000000000000000000000000000000000000
zk_executor_p256_call_success_word=0x0000000000000000000000000000000000000000000000000000000000000001
canonical_base_el_message_passer_slot0=0
zk_executor_message_passer_slot0=1
POC_CONFIRMED: same Azul transaction diverges between canonical Base EL and ZK executor
```

This proves the state-transition mismatch: canonical Base EL leaves the MessagePasser state untouched, while the ZK executor writes the MessagePasser state for the same transaction.

I also verified the canonical Base EL side with a live `base-reth-node --dev` instance. The node activated Azul at genesis:

```
Post-merge hard forks (timestamp based):
- Regolith                         @0
- Canyon                           @0
- Ecotone                          @0
- Fjord                            @0
- Granite                          @0
- Holocene                         @0
- Isthmus                          @0
- Jovian                           @0
- Azul                             @0
```

The live EL behavior matches the expected Azul rules:

```
eth_call to P256VERIFY with total gas 24450
(21000 intrinsic gas + 3450 gas available to the precompile):
out of gas: gas exhausted during precompiled contract execution: 24450

eth_call to P256VERIFY with total gas 27900
(21000 intrinsic gas + 6900 gas available to the precompile):
0x
```

The contract-level `debug_traceCall` shows the internal call to `0x0100` failed before the branch could reach MessagePasser:

```json
{
  "gas": "0xd7a",
  "gasUsed": "0xd7a",
  "to": "0x0000000000000000000000000000000000000100",
  "error": "out of gas",
  "type": "CALL"
}
```

The L1 finalization harness demonstrates the final bridge step after an output root is accepted. It uses Base contracts' `Hashing`, `SecureMerkleTrie`, `Types`, and `SafeCall` libraries, generates a valid MessagePasser withdrawal proof with `scripts/go-ffi/go-ffi`, proves it against an accepted output root, and executes `finalizeWithdrawalTransactionExternalProof`-style logic.

The L1 harness prints:

```
contracts_commit=01dad230390cd69bcf130b5fc7a7a580b31650a7
anvil_chain_id=31337
L1_FINALIZE_HARNESS_CONFIRMED=1
WithdrawalFinalized(..., true)
receiver_calls 1
```

This L1 harness is not intended to redeploy the entire production dispute game or verifier stack. It isolates the bridge finalization mechanics after an output root has been accepted. Combined with the ZK executor mismatch, it shows the impact chain: the ZK executor can create a MessagePasser state that canonical Base EL did not create, and the bridge finalization path can execute withdrawals proven against an accepted MessagePasser root.

## Impact Details

The direct impact is that the ZK proof system can prove the wrong Base Azul state transition function.

This is more severe than a local gas-accounting mismatch. The proof can be valid for the ZK program while the proved output root is not the canonical Base EL output root.

The bridge-loss path is:

1. The attacker deploys an L2 contract with a branch controlled by `CALL(0x0100, gas=3450)`.
2. In the success branch, the contract calls `L2ToL1MessagePasser.initiateWithdrawal(...)`.
3. Canonical Base EL executes under Azul/Osaka rules. The P256 call fails, so no withdrawal is written.
4. The ZK executor executes under the overwritten legacy P256 rule. The P256 call succeeds, so the withdrawal branch executes.
5. The ZK output root can include a `messagePasserStorageRoot` where `sentMessages[withdrawalHash] = true`.
6. The L1 bridge flow verifies withdrawal inclusion against the accepted `messagePasserStorageRoot`.
7. `OptimismPortal2.finalizeWithdrawalTransactionExternalProof(...)` can then execute the L1 target call with the withdrawal value.

This maps to the selected impact because the bridge trusts the accepted output root and verifies withdrawals against the MessagePasser storage root contained in that root.

The relevant L2 contract records withdrawals by storing:

```solidity
sentMessages[withdrawalHash] = true;
```

The L1 portal then proves inclusion of that key/value pair against `_outputRootProof.messagePasserStorageRoot`:

```solidity
SecureMerkleTrie.verifyInclusionProof({
    _key: abi.encode(storageKey),
    _value: hex"01",
    _proof: _withdrawalProof,
    _root: _outputRootProof.messagePasserStorageRoot
})
```

If the accepted root was produced by the incorrect ZK executor, the L1 portal has no independent way to distinguish that the withdrawal was not present in canonical Base EL. It verifies the withdrawal against the accepted root and finalizes it.

## References

* Affected ZK precompile list and overwrite:
  * `crates/proof/succinct/utils/client/src/precompiles/mod.rs`
  * `get_precompiles()` returns legacy `bn254::pair::ISTANBUL` and `secp256r1::P256VERIFY`.
  * `get_or_create_precompiles(OpSpecId::AZUL)` starts from `BasePrecompiles::azul()` and then calls `precompiles.extend(get_precompiles())`.
  * <https://github.com/base/base/blob/dd3b5eb575be368280a5cd4e14be57b78c30712e/crates/proof/succinct/utils/client/src/precompiles/mod.rs#L66-L100>
* Canonical Base Azul precompile table:
  * `crates/common/evm/src/precompiles/provider.rs`
  * `BasePrecompiles::azul()` installs `modexp::OSAKA` and `secp256r1::P256VERIFY_OSAKA`.
  * <https://github.com/base/base/blob/dd3b5eb575be368280a5cd4e14be57b78c30712e/crates/common/evm/src/precompiles/provider.rs#L116-L124>
* ZK EVM factory uses the overwritten provider:
  * `crates/proof/succinct/utils/client/src/precompiles/factory.rs`
  * <https://github.com/base/base/blob/dd3b5eb575be368280a5cd4e14be57b78c30712e/crates/proof/succinct/utils/client/src/precompiles/factory.rs#L45-L58>
* ZK witness executor uses `ZkvmOpEvmFactory::new()`:
  * `crates/proof/succinct/utils/client/src/witness/executor.rs`
  * <https://github.com/base/base/blob/dd3b5eb575be368280a5cd4e14be57b78c30712e/crates/proof/succinct/utils/client/src/witness/executor.rs#L144-L149>
* L2 MessagePasser withdrawal storage:
  * `src/L2/L2ToL1MessagePasser.sol`
  * <https://github.com/base/contracts/blob/v8.1.0/src/L2/L2ToL1MessagePasser.sol#L78-L90>
* L1 withdrawal proof verification and finalization:
  * `src/L1/OptimismPortal2.sol`
  * <https://github.com/base/contracts/blob/v8.1.0/src/L1/OptimismPortal2.sol#L390-L412>
  * <https://github.com/base/contracts/blob/v8.1.0/src/L1/OptimismPortal2.sol#L466-L490>

## Link to Proof of Concept

<https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c>

## Proof of Concept

The PoC is provided as a secret Gist:

`https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c`

To reproduce the primary state-transition mismatch:

```sh
git clone https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c.git base-azul-zk-precompile-poc
cd base-azul-zk-precompile-poc
bash run_poc.sh
```

The script checks out:

```
https://github.com/base/base.git
commit dd3b5eb575be368280a5cd4e14be57b78c30712e
```

The PoC passes when it prints:

```
L1_FINALIZE_HARNESS_CONFIRMED=1
WithdrawalFinalized(..., true)
receiver_calls 1
```

## References

* Secret Gist containing the runnable PoC and logs:
  * <https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c>
* Primary reproduction files:
  * `README.md`: <https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c#file-readme-md>
  * `run_poc.sh`: <https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c#file-run\\_poc-sh>
  * `poc_main.rs`: <https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c#file-poc\\_main-rs>
* Evidence logs and summaries:
  * `zk-executor-message-passer-poc.log`: <https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c#file-zk-executor-message-passer-poc-log>
  * `live-el-summary.md`: <https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c#file-live-el-summary-md>
  * `live-node-rpc-and-trace-results.jsonl`: <https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c#file-live-node-rpc-and-trace-results-jsonl>
  * `l1-finalize-summary.md`: <https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c#file-l1-finalize-summary-md>
  * `l1-finalize-harness.log`: <https://gist.github.com/s-zaizen/420f1caf8965753715be81b32483e56c#file-l1-finalize-harness-log>
