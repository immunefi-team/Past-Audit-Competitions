# 75354 bc medium hardcoded 10 block zk witness interval makes 30 block challenge proofs unverifiable

Submitted on Apr 28th 2026 at 17:46:05 UTC by @Tradi3 for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75354
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Direct loss to Base or users ≥ 10% of funds held within Bridge.

## Description

### Brief/Intro

Base Azul's deployed multiproof verifier is configured for 30-block intermediate checkpoints, but the shipped ZK witness path always builds SP1 stdin with a hardcoded 10-block intermediate-root interval. For a 30-block `challenge()` or `nullify()` proof, `AggregateVerifier` verifies a journal containing exactly one intermediate root, while the official ZK service commits to a journal containing three roots.

That makes the official challenger/prover path produce ZK proofs for different public inputs than the contract verifies. If an invalid TEE proposal needs to be corrected by the ZK fallback path, the bundled service repeatedly generates an unverifiable proof and the invalid proposal can survive the dispute window.

This maps to the Base Azul critical impact: circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization. Because finalized invalid output roots can be used as the basis for bridge withdrawals, this also matches the selected bridge-funds-at-risk impact.

Reviewer shortcut: the secret gist in the PoC section includes `base-azul-source-backed-regression-poc.py`. That script reads the scoped source repos directly and verifies the deployed interval, the hardcoded ZK witness interval, the service/protobuf/challenger request path, and the one-root journal checked by `AggregateVerifier`. It can also be run as `python base-azul-source-backed-regression-poc.py --as-regression-test`, where it exits non-zero while this mismatch is present.

## Vulnerability Details

The deployed Base Azul multiproof parameters are:

* `base/contract-deployments` at `e76fde63ff26d19d47a1f38a746d2b495bc60cd2`
* `sepolia/2026-04-20-activate-multiproof/.env:14-16`
* `BLOCK_INTERVAL=600`
* `INTERMEDIATE_BLOCK_INTERVAL=30`
* `PROOF_THRESHOLD=1`

The verifier uses that 30-block interval on-chain:

* `base/contracts` at `34dbd708c063729caed8d3f5e975963ce458076d`
* `src/multiproof/AggregateVerifier.sol:520` and `:586` pass `abi.encodePacked(intermediateRootToProve)` into proof verification for `challenge()` and `nullify()`. That is a one-root journal for one intermediate checkpoint.
* `src/multiproof/AggregateVerifier.sol:1022-1023` computes the challenged subrange using `INTERMEDIATE_BLOCK_INTERVAL`.

The official ZK prover path does not use that interval:

* `base/base` at `4b389f03b4963ec7bbb4d024445af6281a90d9a8`
* `crates/proof/succinct/utils/client/src/client.rs:19` defines `DEFAULT_INTERMEDIATE_ROOT_INTERVAL: u64 = 10`.
* `crates/proof/succinct/utils/client/src/client.rs:190` records a root when `blocks_processed.is_multiple_of(interval)`.
* `crates/proof/zk/service/src/backends/op_succinct/provider.rs:100-103` passes the hardcoded `DEFAULT_INTERMEDIATE_ROOT_INTERVAL` into `get_sp1_stdin()`.
* `crates/proof/succinct/validity/src/proof_requester.rs:217-219` does the same for proof requests.
* `crates/proof/succinct/programs/range/ethereum/src/main.rs:27` reads the interval from stdin, so the program commits to whatever interval the service supplied.
* `crates/proof/challenge/src/driver.rs:626` builds ZK requests with `number_of_blocks_to_prove: candidate.intermediate_block_interval`, but the request/API has no field that passes the 30-block journal interval through to the ZK service.

The result is deterministic:

* Contract side for a 30-block challenge/nullify proof: `intermediateRoots = root_at_30`, 32 bytes.
* Official ZK service for the same 30-block request: `intermediateRoots = root_at_10 || root_at_20 || root_at_30`, 96 bytes.

The SP1 aggregation path commits to a hash over the packed public journal. `AggregateVerifier` independently constructs the expected packed journal and asks `ZKVerifier.verify()` to verify that digest. Since the official proof and the contract are using different journal bytes, the proof cannot satisfy the verifier.

## Impact

An invalid TEE proposal can require ZK fallback correction. In the shipped stack, the challenger detects the invalid checkpoint and requests a proof for one `candidate.intermediate_block_interval` range. The ZK service then builds the witness with interval 10, creating a proof for the wrong public journal. The `challenge()` or `nullify()` transaction fails proof verification.

Retries do not fix this, because each retry uses the same hardcoded interval. With `PROOF_THRESHOLD=1`, a malicious or faulty TEE proposal only needs its original proof plus time delay to resolve if the ZK correction path cannot submit a valid proof before finalization.

A custom, manually patched prover that uses interval 30 could work around this. The issue is that the official Base Azul challenger/prover stack as shipped cannot exercise the deployed on-chain ZK dispute path for this configuration.

I checked the published known-vulnerability list and did not find this issue. The closest item is "No block range validation on ProveBlock", but this is different: the range length is the intended 30-block challenge interval; the failure is that the public journal interval inside the official ZK witness generation path is hardcoded to 10 while the verifier expects 30.

## Link to Proof of Concept

<https://gist.github.com/krutftw/4cced8aa9a16f46c3253f4c82d70596e>

## Proof of Concept

Secret gist with full report and standalone PoC:

<https://gist.github.com/krutftw/4cced8aa9a16f46c3253f4c82d70596e>

The attached gist contains:

* `base-azul-zk-interval-mismatch-report.md`
* `base-azul-zk-interval-mismatch-poc.py`
* `base-azul-source-backed-regression-poc.py`

## Recommended reviewer PoC

1. Clone the three scoped repositories.
2. Run the source-backed regression PoC:

```bash
python base-azul-source-backed-regression-poc.py \
  --base /path/to/base \
  --contracts /path/to/contracts \
  --deployments /path/to/contract-deployments
```

This script reads the actual scoped source files and deployed config. It verifies:

* deployed `INTERMEDIATE_BLOCK_INTERVAL=30`
* `DEFAULT_INTERMEDIATE_ROOT_INTERVAL=10`
* official ZK service passes the default interval into `get_sp1_stdin()`
* range program reads the interval from stdin
* challenger only passes `number_of_blocks_to_prove = candidate.intermediate_block_interval`
* `ProveBlockRequest` has no `intermediate_root_interval` override
* `AggregateVerifier.challenge()` / `nullify()` verify a one-root checkpoint journal

Expected output from the scoped revisions:

```
Repository heads
  base/base:                 4b389f03b4963ec7bbb4d024445af6281a90d9a8
  base/contracts:            34dbd708c063729caed8d3f5e975963ce458076d
  base/contract-deployments: e76fde63ff26d19d47a1f38a746d2b495bc60cd2

Source-backed facts
  .../.env:14 BLOCK_INTERVAL=600
  .../.env:15 INTERMEDIATE_BLOCK_INTERVAL=30
  .../.env:16 PROOF_THRESHOLD=1
  .../client.rs:19 DEFAULT_INTERMEDIATE_ROOT_INTERVAL=10
  .../client.rs:190 records roots using supplied interval
  .../provider.rs:102 official ZK service passes default interval
  .../proof_requester.rs:219 proof requester passes default interval
  .../main.rs:27 range program reads interval from stdin
  .../driver.rs:626 challenger requests one deployed checkpoint interval
  .../zk_prover.proto:20 ProveBlockRequest has no intermediate_root_interval field
  .../AggregateVerifier.sol:520 challenge/nullify verify a one-root checkpoint journal

Derived public-journal mismatch
  30-block challenge/nullify contract roots: 1 root, 32 bytes, sha256=cdcd08b397c27b21b7ea18b562a44cfd1dc947655b80f2eb109fbb0f2cda0fb3
  30-block challenge/nullify official ZK roots: 3 roots, 96 bytes, sha256=e80658e9d3057dd6bbab4e672c6d707a59609a3a059fbdb81419c12c89afe937
  600-block proposal contract roots: 20 roots, 640 bytes, sha256=acca40aa41124a74aef605356e173326a16da64326946bfceb3c952ad6527882
  600-block proposal official ZK roots: 60 roots, 1920 bytes, sha256=ebb5206a724f747be2f8547034ab61cc89f5d6aa7645150e380e495ba47e5524

Result: VULNERABLE. The official ZK path commits to interval=10 journals,
but the deployed AggregateVerifier verifies interval=30 journals.
```

The same script can be run as a CI regression test with:

```bash
python base-azul-source-backed-regression-poc.py --as-regression-test
```

In regression mode it exits with status `1` while the vulnerable mismatch is present.

\### Minimal standalone model

The second PoC models only public-input construction. It does not need to run SP1, because the failure is before execution correctness: the proof is created for a different public journal than the verifier checks.

Steps:

1. Download the PoC from the secret gist.
2. Run `python base-azul-zk-interval-mismatch-poc.py`.
3. Observe that the deployed contract journal and official ZK service journal differ for both a 30-block challenge/nullify proof and a 600-block proposal proof.

Expected output:

```
Contract journal for a 30-block challenge/nullify proof:
  roots: 1
  bytes: 32
  packed sha256: cdcd08b397c27b21b7ea18b562a44cfd1dc947655b80f2eb109fbb0f2cda0fb3
  sampled relative blocks: [30]
Official ZK service journal for the same 30-block request:
  roots: 3
  bytes: 96
  packed sha256: e80658e9d3057dd6bbab4e672c6d707a59609a3a059fbdb81419c12c89afe937
  sampled relative blocks: [10, 20, 30]

Contract journal for a 600-block proposal proof:
  roots: 20
  bytes: 640
  packed sha256: acca40aa41124a74aef605356e173326a16da64326946bfceb3c952ad6527882
  sampled relative blocks: [30, 60, 90, 120, 150, 180, 210, 240, 270, 300, 330, 360, 390, 420, 450, 480, 510, 540, 570, 600]
Official ZK service journal for the same 600-block request:
  roots: 60
  bytes: 1920
  packed sha256: ebb5206a724f747be2f8547034ab61cc89f5d6aa7645150e380e495ba47e5524
  sampled relative blocks: [10, 20, 30, 40, 50, 60, 70, 80, 90, 100, 110, 120, 130, 140, 150, 160, 170, 180, 190, 200, 210, 220, 230, 240, 250, 260, 270, 280, 290, 300, 310, 320, 330, 340, 350, 360, 370, 380, 390, 400, 410, 420, 430, 440, 450, 460, 470, 480, 490, 500, 510, 520, 530, 540, 550, 560, 570, 580, 590, 600]

Result: public-input mismatch reproduced. Official ZK proofs are built for interval=10,
but the deployed AggregateVerifier verifies journals for interval=30.
```

Why this proves the issue:

* `AggregateVerifier.challenge()` / `nullify()` verify one deployed intermediate checkpoint, so the contract-side public input contains one root for a 30-block proof.
* The official ZK service passes `DEFAULT_INTERMEDIATE_ROOT_INTERVAL = 10` into SP1 stdin, so the proof commits to three roots for the same 30-block proof request.
* A proof generated for the 96-byte journal cannot verify against the contract's 32-byte journal digest.

The same mismatch exists for full proposal proofs: the contract expects 20 intermediate roots for 600/30, while the official ZK path emits 60 roots for 600/10.
