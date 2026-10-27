# 75616 bc medium base v1 succinct zkvm p256 precompile override produces divergent state and output roots

## #75616 \[BC-Medium] Base V1 Succinct ZKVM P256 Precompile Override Produces Divergent State and Output Roots

**Submitted on Apr 30th 2026 at 04:19:31 UTC by @SecureHedgehog6828 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75616
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

### Description

## Base V1 Succinct ZKVM P256 Precompile Override Produces Divergent State and Output Roots

Severity: High

Requested impact category: High — Unintended chain split / proof-route finality divergence

Affected component: `base/base`, Base-owned Succinct ZKVM execution integration under `crates/succinct/utils/client`

### Summary

Base V1 host execution installs the Osaka P256 precompile at address `0x0100`, but the Base-owned Succinct ZKVM execution integration can replace that Base V1 entry with the legacy P256 precompile at the same address.

The host path builds Base V1 precompiles through `BasePrecompiles::base_v1()`, which includes `secp256r1::P256VERIFY_OSAKA` (`crates/common/evm/src/precompiles.rs:119-127`). The Succinct ZKVM provider also starts from `BasePrecompiles::base_v1()` for `OpSpecId::BASE_V1`, but then extends the set with a generic accelerated precompile set that includes `secp256r1::P256VERIFY`, the legacy P256 implementation (`crates/succinct/utils/client/src/precompiles/mod.rs:65-73`, `:90-100`). Because both variants are registered at `0x0100`, the ZKVM execution path can use the legacy 3,450 gas P256 behavior where the Base V1 host path expects the Osaka 6,900 gas behavior.

A normal L2 transaction can exploit this gas-cost mismatch by calling `0x0100` with a gas stipend that is sufficient for legacy P256 but insufficient for Osaka P256. The PoC uses a 5,000 gas call and branches on the EVM `CALL` success bit. Host Base V1 execution returns the failure branch due to out-of-gas, while the Succinct ZKVM EVM factory returns the success branch because the legacy P256 call returns normally.

A follow-up stateless-builder harness demonstrates root-level impact: swapping only the host EVM factory versus the Succinct ZKVM EVM factory produces different post-state roots and output roots for the same transaction/context. This report is scoped to the Base-owned Succinct integration and does not claim accepted invalid state, L1 finalization, mainnet impact, fund loss, withdrawal bypass, or an upstream SP1/core prover/circuit bug.

### Impact

Impact category requested: High — Unintended chain split / proof-route finality divergence.

A normal L2 transaction can make the Base V1 host executor and the Base-owned Succinct ZKVM execution integration derive different execution results for the same block context. In the PoC, a contract calls P256 address `0x0100` with 5,000 gas and branches on the EVM `CALL` success bit. Host Base V1 execution uses Osaka P256 pricing, so the call is out of gas and the contract takes the failure branch. The Succinct ZKVM EVM factory uses the legacy P256 path, so the same call returns normally and the contract takes the success branch.

The stronger stateless-builder harness demonstrates that this is not only a return-data difference. With the same transaction/context, swapping only the host versus Succinct ZKVM EVM factory produces different post-state roots and output roots:

```
host_state_root=0xe260af35599683a799ccbe3b2607371f7c43baf421e4445cbafd7dd157275176
host_output_root=0xede587ff9aa2c5c8d4bf091aff84caee40c28c7508f832b278d749b56dbb6762
host_gas_used=48246

zkvm_state_root=0x18784824c56ad57af42c27d2e41a99b9c5fb62c5e096421355120e297182d15a
zkvm_output_root=0x543d7e8510b51be362ade6fe21671a47b6d7b26050cad4f7515ca19934e4e4bb
zkvm_gas_used=46697
```

This creates a High-severity proof-route finality/chain-split risk for the in-scope Base V1 environment because the proof executor can commit to an execution result that does not match host Base V1 execution. The report does not claim that an invalid state was accepted by the verifier, does not claim L1 finalization, does not claim Base mainnet impact, and does not claim fund loss or withdrawal bypass.

### Affected Path

Host execution installs Base V1 precompiles through `BasePrecompiles::new_with_spec`:

* `crates/common/evm/src/precompiles.rs:30-43`
* `crates/common/evm/src/precompiles.rs:119-127`

The Succinct ZKVM EVM factory installs `OpZkvmPrecompiles`:

* `crates/succinct/utils/client/src/precompiles/factory.rs:33-56`

The ZKVM provider selects Base V1 and then applies the generic accelerated P256 override:

* `crates/succinct/utils/client/src/precompiles/mod.rs:65-73`
* `crates/succinct/utils/client/src/precompiles/mod.rs:90-100`
* `crates/succinct/utils/client/src/precompiles/mod.rs:184-195`

The production witness path injects this factory into the proof executor:

* `crates/succinct/utils/client/src/witness/executor.rs:144-149`
* `crates/proof/proof/src/executor.rs:77-95`
* `crates/proof/executor/src/builder/core.rs:74-87`
* `crates/proof/executor/src/builder/core.rs:102-160`
* `crates/proof/executor/src/builder/env.rs:53-58`

Base V1 is configured for Base Sepolia at timestamp `1776708000`, while mainnet is not activated at this checked revision:

* `docs/specs/pages/upgrades/v1/overview.md:15-20`
* `crates/common/chains/src/config.rs:156-168`
* `crates/common/chains/src/config.rs:220-235`

The repository contains checked-in Sepolia deployment artifacts for the ZK verifier configuration update; these are cited only as deployment-context artifacts, not as a live RPC verification claim:

* `contract-deployments/sepolia/2026-04-23-update-zk-config/README.md:1-11`
* `contract-deployments/sepolia/2026-04-23-update-zk-config/addresses.json:1-4`
* `contracts/src/multiproof/AggregateVerifier.sol:917-932`

### Link to Proof of Concept

<https://gist.github.com/hellozenstrategist-lab/12c5c381b43d35a594359baa9ae30d75>

### Proof of Concept

The runnable PoC patch is provided through the secret Gist linked in this report because Immunefi evidence uploads only support image formats. Open the Gist, copy/apply `BASE_AZUL_OSAKA_ZKVM_P256_POC_PATCH_20260429.diff` to a local checkout of `base/base` at `v0.8.0-rc.28`, then run the commands below. After applying the patch, the added tests are located in `crates/succinct/utils/client/src/precompiles/mod.rs`.

The local harness adds the test:

```
precompiles::tests::test_base_v1_p256_branch_transaction_diverges_between_host_and_zkvm_factories
```

The test constructs one Base V1 transaction and executes it twice through the same `BaseBlockExecutorFactory` / `BaseBlockExecutor` layer:

* once with `BaseEvmFactory::default()`
* once with `ZkvmOpEvmFactory::new()`

The contract bytecode calls P256 address `0x0100` with exactly `5,000` gas and branches on the EVM `CALL` success bit:

```
60006000600060006000610100611388f1602457664641494c55524560005260206000f35b665355434345535360005260206000f3
```

The shared harness context uses a synthetic Base V1 rollup config with `OpSpecId::BASE_V1` active. The chain ID is not material to the precompile-provider mismatch.

Test parameters:

* transaction gas limit: `100,000`
* P256 call gas: `5,000`
* host path: `BaseEvmFactory::default()`
* ZKVM path: `ZkvmOpEvmFactory::new()`

The key assertion is:

```
host CALL success bit: 0 / returned marker: FAILURE
zkvm CALL success bit: 1 / returned marker: SUCCESS
```

The branch is on the EVM `CALL` success bit caused by gas accounting, not on the cryptographic validity of a P256 signature.

A stronger stateless-builder harness also measured root-level divergence for the same transaction/context:

```
host_state_root=0xe260af35599683a799ccbe3b2607371f7c43baf421e4445cbafd7dd157275176
host_output_root=0xede587ff9aa2c5c8d4bf091aff84caee40c28c7508f832b278d749b56dbb6762
host_gas_used=48246

zkvm_state_root=0x18784824c56ad57af42c27d2e41a99b9c5fb62c5e096421355120e297182d15a
zkvm_output_root=0x543d7e8510b51be362ade6fe21671a47b6d7b26050cad4f7515ca19934e4e4bb
zkvm_gas_used=46697
```

### Reproduction

From the Base repository root:

```bash
cd <base-repo-root>
RUSTFLAGS='' cargo test -p base-succinct-client-utils test_base_v1_p256_branch_transaction_diverges_between_host_and_zkvm_factories -- --nocapture
RUSTFLAGS='' cargo test -p base-succinct-client-utils test_stateless_builder_p256_branch_state_and_output_roots_diverge_between_host_and_zkvm -- --nocapture
```

Expected result for the executor-level divergence test:

```
running 1 test
test precompiles::tests::test_base_v1_p256_branch_transaction_diverges_between_host_and_zkvm_factories ... ok

test result: ok. 1 passed; 0 failed
```

Expected result for the stateless-builder root-divergence test:

```
running 1 test
test precompiles::tests::test_stateless_builder_p256_branch_state_and_output_roots_diverge_between_host_and_zkvm ... ok

test result: ok. 1 passed; 0 failed
```

Additional reported validation commands:

```bash
RUSTFLAGS='' cargo test -p base-succinct-client-utils precompiles::tests -- --nocapture
RUSTFLAGS='' cargo test -p base-succinct-client-utils
git diff --check -- crates/succinct/utils/client/Cargo.toml crates/succinct/utils/client/src/precompiles/mod.rs
```

Expected results:

```
precompiles::tests: 16 passed
base-succinct-client-utils: 17 passed
git diff --check: no output
```

### Residuals / Non-Claims

* No full SP1 proof was generated. This report claims a pre-proof execution integration divergence in the Base-owned Succinct ZKVM EVM factory path, not accepted verifier output.
* No live RPC was used. Checked-in Sepolia deployment artifacts are cited only as repository deployment-context artifacts.
* The PoC uses synthetic state, contract bytecode, and payload data, but exercises the production block executor/factory path and a stateless-builder path.
* The stateless-builder harness measures divergent post-state roots and output roots locally for the same transaction/context.
* This report does not claim accepted invalid state, L1 finalization, Base mainnet impact, fund loss, withdrawal bypass, TEE bypass, or verifier-contract compromise.
* This report does not frame the issue as an upstream SP1/core prover/circuit bug. The root cause is the Base-owned Succinct integration under `crates/succinct/utils/client`.
* BN254 is intentionally excluded from this submission.

### Recommended Fix

Do not let the generic ZKVM accelerated P256 precompile overwrite the Base V1 P256 entry. For `OpSpecId::BASE_V1`, the ZKVM provider should preserve `BasePrecompiles::base_v1()` for P256, or explicitly install the Osaka P256 implementation for address `0x0100`.

Add parity tests asserting that host `BasePrecompiles::base_v1()` and `OpZkvmPrecompiles::new_with_spec(OpSpecId::BASE_V1)` both apply Osaka P256 gas behavior for address `0x0100`, including an out-of-gas result for a 5,000 gas P256 call under Base V1.
