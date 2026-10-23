# 75230 bc medium zk prover service hardcodes default intermediate root interval 10 while on chain intermediate block interval 30 causing journal digest mismatch that blocks all zk challenge proofs

**Submitted on Apr 27th 2026 at 22:14:53 UTC by @Meadowlark14896 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75230
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Summary

The ZK prover service hardcodes `DEFAULT_INTERMEDIATE_ROOT_INTERVAL = 10` when generating witnesses for ZK proofs, but the on-chain `AggregateVerifier` is deployed with `INTERMEDIATE_BLOCK_INTERVAL = 30`. This mismatch causes the ZK proof's journal digest to differ from the on-chain expected digest, making all ZK challenge proofs fail verification. A malicious proposer can submit incorrect intermediate roots that can never be corrected through the dispute mechanism.

## Root Cause

In `crates/proof/zk/service/src/backends/op_succinct/provider.rs` (lines 100-103), the witness generation hardcodes the intermediate root interval:

```rust
let stdin = self.host.witness_generator().get_sp1_stdin(
    witness,
    base_succinct_client_utils::client::DEFAULT_INTERMEDIATE_ROOT_INTERVAL, // = 10
)?;
```

Where `DEFAULT_INTERMEDIATE_ROOT_INTERVAL` is defined in `crates/succinct/utils/client/src/client.rs:19`:

```rust
pub const DEFAULT_INTERMEDIATE_ROOT_INTERVAL: u64 = 10;
```

The on-chain deployment uses `INTERMEDIATE_BLOCK_INTERVAL = 30` (from `contract-deployments/sepolia/2026-04-20-activate-multiproof/.env`).

The `ProveBlockRequest` protobuf (`crates/proof/zk/client/proto/prover.proto`) has no `intermediate_root_interval` field, so the challenger cannot override this value.

## How the Mismatch Breaks Challenges

When the challenger detects an incorrect intermediate root and initiates a ZK challenge proof for a 30-block range:

**Off-chain (ZK prover service with interval=10):**

* Range program runs over 30 blocks with `intermediate_root_interval = 10`
* Produces 3 intermediate roots (at blocks start+10, start+20, start+30)
* Aggregation output: `intermediateRoots = 3 * 32 = 96 bytes`

**On-chain (AggregateVerifier.challenge() with interval=30):**

* Constructs journal with `intermediateRoots = abi.encodePacked(intermediateRootToProve) = 32 bytes`

**Result:** 96 bytes != 32 bytes -> different keccak256 digests -> `ZK_VERIFIER.verify()` returns false -> revert `InvalidProof()`

## Attack Scenario

{% stepper %}
{% step %}

## Proposer creates a game with TEE proof containing 20 intermediate roots (600 blocks / 30 interval)

One intermediate root at index `i` is incorrect.
{% endstep %}

{% step %}

## Challenger detects the mismatch via independent validation

The challenger identifies the incorrect intermediate root.
{% endstep %}

{% step %}

## Challenger calls the ZK prover service to generate a challenge proof for the 30-block range at index `i`

A ZK challenge proof is requested for the disputed range.
{% endstep %}

{% step %}

## ZK prover service generates witness with `intermediate_root_interval = 10` (hardcoded, cannot be overridden)

The service uses the default interval instead of the on-chain interval.
{% endstep %}

{% step %}

## Range program produces 3 intermediate roots for the 30-block range

The proof is generated with the wrong interval.
{% endstep %}

{% step %}

## Aggregation output has 96 bytes of intermediate roots

The off-chain proof commits to 96 bytes.
{% endstep %}

{% step %}

## On-chain `challenge()` constructs journal with 32 bytes of intermediate roots

The on-chain verifier expects 32 bytes.
{% endstep %}

{% step %}

## `ZK_VERIFIER.verify()` fails due to digest mismatch

The journal digest does not match the committed proof.
{% endstep %}

{% step %}

## Challenge reverts with `InvalidProof()`

The proof cannot be accepted.
{% endstep %}

{% step %}

## The incorrect intermediate root cannot be corrected

The dispute path is blocked.
{% endstep %}

{% step %}

## Game resolves with `DEFENDER_WINS` despite having an incorrect state root

The dispute mechanism fails to correct the wrong root.
{% endstep %}
{% endstepper %}

## Impact

The dispute/challenge mechanism is rendered non-functional for ZK proofs. Without working ZK challenges, incorrect TEE proofs cannot be disputed, the soundness alert mechanism cannot trigger, and the multiproof security model (TEE + ZK) degrades to TEE-only.

## Code References

| Component                    | File                                                           | Line    | Value                                      |
| ---------------------------- | -------------------------------------------------------------- | ------- | ------------------------------------------ |
| ZK service hardcode          | `crates/proof/zk/service/src/backends/op_succinct/provider.rs` | 101-103 | Uses DEFAULT\_INTERMEDIATE\_ROOT\_INTERVAL |
| Default constant             | `crates/succinct/utils/client/src/client.rs`                   | 19      | = 10                                       |
| Protobuf (no interval field) | `crates/proof/zk/client/proto/prover.proto`                    | -       | No intermediate\_root\_interval field      |
| Range program reads interval | `crates/succinct/programs/range/ethereum/src/main.rs`          | 28      | sp1\_zkvm::flag\_io::read()                |
| On-chain challenge journal   | `src/multiproof/AggregateVerifier.sol`                         | 518-520 | abi.encodePacked(intermediateRootToProve)  |
| Deployed config              | `contract-deployments/sepolia/.env`                            | -       | INTERMEDIATE\_BLOCK\_INTERVAL=30           |

## Why This Is Not a Duplicate

* Not in the Known Vulnerabilities PDF (no mention of "intermediate root interval" or "DEFAULT\_INTERMEDIATE\_ROOT\_INTERVAL" or "interval mismatch")
* Not the SP1 soundness issue (GHSA-63x8-x938-vx33)
* Not "No Block Range Validation on ProveBlock" (that's about integer overflow)
* Not "Proposer Does Not Independently Recompute Output Roots" (that's about the proposer, not the challenge mechanism)

## Recommendation

Add an `intermediate_root_interval` field to the `ProveBlockRequest` protobuf so callers can specify the correct interval, and use it in `provider.rs` instead of the hardcoded default.

## Link to Proof of Concept

<https://gist.github.com/bigfootbjj/c042c8d390aa3a854a482816e5f02312>

## Proof of Concept

Foundry PoC: `test/multiproof/IntermediateRootMismatch_PoC.t.sol`

Run: `forge test --match-test "test_journalDigestMismatch" -vvv`

Result (PASS):

* On-chain roots: 32 bytes
* Off-chain roots: 96 bytes
* RESULT: Digests differ -> challenge blocked

The test computes two journal digests using the same fields except `intermediateRoots`:

1. On-chain path (interval=30): `abi.encodePacked(ROOT_AT_30)` = 32 bytes\
   This is what `AggregateVerifier.challenge()` constructs at line 520.
2. Off-chain path (interval=10): `abi.encodePacked(ROOT_AT_10, ROOT_AT_20, ROOT_AT_30)` = 96 bytes\
   This is what the ZK proof commits, because `provider.rs` hardcodes `DEFAULT_INTERMEDIATE_ROOT_INTERVAL=10` (`client.rs:19`) for a 30-block range.

The keccak256 digests differ, so `ZK_VERIFIER.verify(proof, ZK_AGGREGATE_HASH, onChainDigest)` returns false because the proof was committed against `offChainDigest`.

`challenge()` reverts with `InvalidProof()`. The dispute mechanism is non-functional.

Code path:

* Challenger -> `build_zk_request` (`driver.rs:614-629`) -> `ProveBlockRequest { number_of_blocks_to_prove: 30 }` -> ZK prover gRPC -> `OpSuccinctProvider::generate_witness` (`provider.rs:47-108`) -> `get_sp1_stdin(witness, DEFAULT_INTERMEDIATE_ROOT_INTERVAL=10)` // line 101-103 -> Range program runs with interval=10, produces 3 roots for 30 blocks -> Aggregation commits digest with 96 bytes of `intermediateRoots`
* On-chain `challenge()` (`AggregateVerifier.sol:505-520`) -> `journal = keccak256(..., abi.encodePacked(intermediateRootToProve), ...)` // 32 bytes -> `ZK_VERIFIER.verify(proof, ZK_AGGREGATE_HASH, journal)` -> FALSE -> revert `InvalidProof()`

Gist: \[COLLE L'URL DU GIST ICI]
