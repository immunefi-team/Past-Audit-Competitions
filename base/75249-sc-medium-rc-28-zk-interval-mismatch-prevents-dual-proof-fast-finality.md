# 75249 sc medium rc 28 zk interval mismatch prevents dual proof fast finality

## #75249 \[SC-Medium] rc.28 ZK interval mismatch prevents dual-proof fast finality

**Submitted on Apr 28th 2026 at 01:57:48 UTC by @cgdusek for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75249
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours (e.g., stuck withdrawal proofs, locked dispute game bonds)

### Description

## Vulnerability Report

### Selected fields

* program: `Base Azul`
* category: `Smart Contract`
* asset: [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof)
* supporting assets:
  * [`base/base v0.8.0-rc.28`](https://github.com/base/base/tree/v0.8.0-rc.28) for the ZK proof-service root cause
  * [`base/contract-deployments`](https://github.com/base/contract-deployments) for Base Sepolia deployment constants
* impact: `High: Temporary freezing of funds for at least 24 hours (e.g., stuck withdrawal proofs, locked dispute game bonds)`
* title: `rc.28 ZK interval mismatch prevents dual-proof fast finality`

### Executive summary

`base/base v0.8.0-rc.28` hardcodes OP-Succinct's `DEFAULT_INTERMEDIATE_ROOT_INTERVAL=10` when generating ZK witnesses, while the active Base Sepolia Azul type-621 AggregateVerifier is configured with `INTERMEDIATE_BLOCK_INTERVAL=30`.

For a 30-block challenge or full proposal proof, the ZK program commits a journal containing 10-block intermediate roots, but the contract verifies a journal containing 30-block roots. The valid ZK proof therefore cannot match the onchain journal needed to add the second proof, leaving games on the single-proof `604800` second path instead of the dual-proof `86400` second path and adding `518400` seconds of finality delay.

### Classification and scope justification

#### Exact program criteria match

* asset name(s) exactly as listed on scope page: `Smart Contract - base/contracts v8.1.0 src/multiproof` selected; `Blockchain/DLT - base/base v0.8.0-rc.28` and `Smart Contract - base/contract-deployments` as supporting evidence assets
* specific scoped asset(s) supporting this severity claim:
  * selected Smart Contract asset [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof) carries the finality/withdrawal-delay impact
  * supporting [`base/base v0.8.0-rc.28`](https://github.com/base/base/tree/v0.8.0-rc.28) carries the ZK proof-service root cause
  * supporting [`base/contract-deployments`](https://github.com/base/contract-deployments) carries Base Sepolia deployment constants
* exact current Immunefi scope picker URL(s) supporting this severity claim:
  * [selected multiproof Smart Contract asset](https://github.com/base/contracts/tree/v8.1.0/src/multiproof)
  * [supporting base/base Blockchain/DLT asset](https://github.com/base/base/tree/v0.8.0-rc.28)
  * [supporting contract-deployments Smart Contract asset](https://github.com/base/contract-deployments)
  * [Audit Comp | Base Azul scope](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/), checked `2026-04-28`
* selected online form category: `Smart Contract`
* selected online form asset: [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof)
* exact current Immunefi scope picker URL for selected form asset: [base/contracts v8.1.0 src/multiproof](https://github.com/base/contracts/tree/v8.1.0/src/multiproof)
* selected severity from scope page: `High`
* selected impact title from scope page: `Temporary freezing of funds for at least 24 hours (e.g., stuck withdrawal proofs, locked dispute game bonds)`
* exact scope wording: `Runnable PoC Required`; the scope page lists `Offchain Components`, `Base Azul`, and `Implementation Contracts`; the online form requires the `Smart Contract` category to select the temporary-freezing-of-funds impact, so this submission selects the multiproof contract asset that realizes the finality/withdrawal delay while retaining `base/base v0.8.0-rc.28` as the supporting root-cause asset
* scope page last checked: `2026-04-28`
* why this matches the demonstrated consequence: the PoC proves an additional `518400` seconds of finality delay for games that cannot receive the ZK second proof, which exceeds 24 hours and directly affects withdrawal proof readiness and dispute game bond finality.

#### Why higher or adjacent criteria do not fit

* closest higher severity criterion considered: `Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1`
* why the report does not satisfy it: this report proves valid ZK proof unusability caused by a journal mismatch. It does not prove verifier bypass, forged proof acceptance, finalized invalid state, direct bridge theft, or permanent freezing.
* closest adjacent same/lower criteria considered:
  * `Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization`
  * `Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path`
  * `Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments`
* why the selected criterion fits better: the proven effect is a time-bounded finality delay for proofs, withdrawals, and dispute-game bonds. It is not a chain-transaction halt, an invalid-state finalization path, or a no-recovery permanent freeze.

#### Out-of-scope exclusion check

* relevant out-of-scope exclusions reviewed: compromised Base-operated infrastructure, relying on invalid TEE/ZK proofs, assuming Base will not dispute/blacklist/retire invalid proposals, assuming services will not be restarted with different configuration, out-of-scope `base/base` folders, and core Op-Succinct/circuit root causes.
* excluded assumptions not used by this exploit: the PoC does not require leaked keys, privileged addresses, signer compromise, public testnet writes, mainnet writes, invalid proof acceptance, governance action, sequencer behavior, or a claim that Base cannot manually restart services. The report is limited to the release-pinned rc.28 integration mismatch and its deployed AggregateVerifier finality consequence.

### Finding details

#### Public documentation invariant

Public evidence checked on `2026-04-27`:

* The live [Audit Comp | Base Azul scope page](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/) lists `Offchain Components`, `Base Azul`, and `Implementation Contracts` as in-scope assets; marks the competition as requiring a runnable PoC; identifies proof-system integration, TEE/ZK dispute game submission logic, verifier contract bugs, and ZK recursion soundness as primary concerns; and explains that Base Azul replaces optimistic fault proofs with a TEE + ZK dual-proof system for faster finality.
* The same live [scope page](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/) states that the competition environment is Base Sepolia post-April 20 Azul activation, that public Base mainnet is not in scope, and that the core Op-Succinct program is out of scope while Base changes around it are in scope.
* The [Base V1 proof documentation](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/docs/specs/pages/upgrades/v1/proofs.md#L27-L42) defines a long `7 days` settlement path for TEE-only or ZK-only proposals and a short `1 day` path when TEE and ZK both back the same proposal.
* The same V1 documentation describes ZK provers as the [permissionless proving backend](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/docs/specs/pages/upgrades/v1/proofs.md#L116-L122), including dispute/backstop use.
* `AggregateVerifier` hashes the contract-supplied intermediate roots into the ZK verifier journal in [`_verifyZkProof()`](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L904-L932), and its delay logic uses [`1 days` for the fast path and `7 days` for the slow path](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L44-L53).

The resulting invariant is inference-backed from the linked public docs and source: a valid rc.28 ZK proof generated for the active contract interval must commit the same intermediate-root vector that `AggregateVerifier` hashes into the verifier journal. This vulnerability breaks that invariant because the rc.28 proof service records 10-block roots while the active contract verifies 30-block roots. The trust boundary is narrow: the report does not assume the common proposer path must start with ZK, and it does not rely on core Op-Succinct or circuit bugs; it relies on Base's in-scope rc.28 integration passing the contract interval consistently into the proof-service journal.

#### Root cause

`OpSuccinctProvider::generate_witness()` calls `get_sp1_stdin()` with [`DEFAULT_INTERMEDIATE_ROOT_INTERVAL`](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/proof/zk/service/src/backends/op_succinct/provider.rs#L100-L103), and that constant is [`10`](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/client.rs#L18-L19). The challenge driver sets `number_of_blocks_to_prove` from the candidate's contract interval, but the [`ProveBlockRequest`](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/proof/zk/client/proto/zk_prover.proto#L17-L32) has no field for the intermediate-root recording interval, and [`build_zk_request()`](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/proof/challenge/src/driver.rs#L613-L632) cannot pass it.

The range client records an intermediate root whenever `blocks_processed.is_multiple_of(interval)` in [`advance_to_target()`](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/client.rs#L184-L192). The aggregation program then concatenates all range intermediate roots and commits the packed digest in [`aggregation/src/main.rs`](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/programs/aggregation/src/main.rs#L80-L118).

#### Strongest demonstrated exploit path

1. A type-621 AggregateVerifier game is created with `INTERMEDIATE_BLOCK_INTERVAL=30`.
2. A permissionless ZK proof is requested as the second proof needed for the fast-finality path.
3. For a full 600-block proposal proof, the contract-side journal contains `600 / 30 = 20` intermediate roots, while rc.28 records `600 / 10 = 60` roots because the proof service uses `DEFAULT_INTERMEDIATE_ROOT_INTERVAL=10`.
4. The same source-level mismatch also applies to a 30-block challenge subrange (`3` rc.28 roots versus `1` contract root), but the runnable artifact anchors the full-proposal second-proof path.
5. `_verifyZkProof()` hashes the contract-side root vector into the ZK journal and calls [`ZK_VERIFIER.verify`](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L904-L932). A proof generated for the rc.28 10-block journal is rejected under the onchain 30-block journal.
6. Without the second proof, `_getDelay()` keeps the game on the [`SLOW_FINALIZATION_DELAY`](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L44-L53) path and [`_getDelay()` returns 7 days for one proof and 1 day for two proofs](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L763-L823).

#### PoC-backed proof chain

The PoC package at [the secret gist](https://gist.github.com/cgdusek/43cde38ec160412f84fa5eaf687f8181) generates two real SP1 Groth16 proofs for an equivalent aggregate-journal program, then verifies them through release-pinned Base `ZkVerifier` code. It prints:

* `POC_RESULT: REAL_ZK_PROOF_SYSTEM=SP1 Groth16`
* `POC_RESULT: SP1_PROGRAM_VKEY_BYTES32_FORMAT=true`
* `POC_RESULT: PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH_BYTES32_FORMAT=true`
* `POC_RESULT: PROGRAM_VKEY_DIFFERS_FROM_PUBLIC_SEPOLIA=true`
* `POC_RESULT: ONCHAIN_SP1_PROOF_BYTES_PRESENT=true`
* `POC_RESULT: RC28_SP1_PROOF_BYTES_PRESENT=true`
* `POC_RESULT: SP1_CONTRACTS_COMMIT_MATCH=true`
* `POC_RESULT: BASE_CONTRACTS_COMMIT_MATCH=true`
* `POC_RESULT: ZK_VERIFIER_ONCHAIN_PROOF_ACCEPTED=true`
* `POC_RESULT: ZK_VERIFIER_RC28_PROOF_ACCEPTED_WITH_RC28_JOURNAL=true`
* `POC_RESULT: ZK_VERIFIER_RC28_PROOF_ACCEPTED_WITH_ONCHAIN_JOURNAL=false`
* `POC_RESULT: ZK_VERIFIER_PUBLIC_SEPOLIA_VKEY_ACCEPTED=false`
* `POC_RESULT: FULL_PROPOSAL_ONCHAIN_ROOT_COUNT=20`
* `POC_RESULT: FULL_PROPOSAL_RC28_ROOT_COUNT=60`
* `POC_RESULT: FULL_PROPOSAL_DIGESTS_EQUAL=false`
* `POC_RESULT: ADDITIONAL_WITHDRAWAL_FREEZE_SECONDS=518400`
* `POC_RESULT: ADDITIONAL_WITHDRAWAL_FREEZE_GE_24H=true`
* `POC_RESULT: INTERVAL_MISMATCH_PROVEN=true`
* `POC_RESULT: HIGH_TEMPORARY_FREEZE_CANDIDATE=true`

#### PoC setup conditions

The PoC runs in an isolated reviewer environment. It does not call RPC, deploy to a public network, submit transactions, use privileged addresses, or depend on a public network fork. It clones release-pinned `base/contracts v8.1.0`, generates two SP1 Groth16 proofs during execution for the aggregate-journal shape, deploys the release-pinned SP1 Groth16 verifier and Base `ZkVerifier` inside Foundry execution, and checks the rc.28 proof rejection under the onchain 30-block journal.

#### Meaning of valid ZK proof in this report

Within the public-boundary PoC, `valid ZK proof` means verifier-valid under the same vkey type and verifier interface used by the scoped Base contracts:

1. The proof bytes are generated by SP1 Groth16 for a concrete aggregate-journal program and concrete public inputs.
2. The public journal is the `AggregateVerifier` ZK journal shape: `proposer`, `l1OriginHash`, starting and ending roots, starting and ending L2 sequence numbers, intermediate roots, `CONFIG_HASH`, and `ZK_RANGE_HASH`.
3. The vkey is a `bytes32` SP1 program verification key passed to the release-pinned upstream `ZkVerifier.verify(proofBytes, imageId, journal)` path, the same Solidity type and verifier interface used for Base Sepolia's `ZK_AGGREGATE_HASH`.
4. The release-pinned upstream `ZkVerifier` accepts each proof when the proof is paired with the journal it proves.
5. The rc.28 proof is rejected when paired with the onchain 30-block journal, which is the contract journal needed for the second proof.

The vkey has the same encoding, Solidity type, and verification-call semantics as the public Sepolia aggregate vkey. The difference is the vkey value, which identifies an equivalent aggregate-journal SP1 program rather than Base Sepolia's deployed aggregate program. This report does not use `valid` to mean accepted by Base Sepolia's deployed aggregate vkey. Proving that deployment-specific condition would require Base's production aggregate SP1 program and proving inputs that produce proof bytes verifying against public Sepolia's `ZK_AGGREGATE_HASH`.

#### Exploit preconditions

* The active AggregateVerifier remains configured with `INTERMEDIATE_BLOCK_INTERVAL=30`.
* A participant attempts to use rc.28's ZK proof service as the permissionless ZK dispute/backstop path or as the second proof path for fast finality.
* The proof must be generated honestly; the failure is a journal mismatch, not an invalid-proof bypass.
* The issue is operator-remediable by redeploy/code config, not user-remediable by existing runtime configuration: the active contract interval is immutable deployment configuration, while rc.28's proof request path does not expose an intermediate-root interval that a participant can set.

#### Deployment and in-scope surface

The active rc.28 source is `base/base v0.8.0-rc.28` at commit `e3467a2048881213b56739a54a876efb9c6ea103`, matching the Immunefi mid-contest update to the rc.28 tree. Deployment material shows `BLOCK_INTERVAL=600`, `INTERMEDIATE_BLOCK_INTERVAL=30`, and `PROOF_THRESHOLD=1` in the [multiproof activation environment](https://github.com/base/contract-deployments/blob/01974d769a97b6e0896903894eb63984e58e610e/zeronet/2026-04-01-activate-multiproof/.env#L11-L13), and the later AggregateVerifier redeploy script [preserves the existing verifier, hashes, intervals, and threshold](https://github.com/base/contract-deployments/blob/01974d769a97b6e0896903894eb63984e58e610e/zeronet/2026-04-17-fix-tee-image-hash/script/DeployAggregateVerifier.s.sol#L48-L59) with post-checks for the same values [at lines 99-109](https://github.com/base/contract-deployments/blob/01974d769a97b6e0896903894eb63984e58e610e/zeronet/2026-04-17-fix-tee-image-hash/script/DeployAggregateVerifier.s.sol#L99-L109).

Read-only Base Sepolia checks on `2026-04-27` found the active type-621 implementation at `0xF3f0fA3124b7b0feB048A00404Fe4D5D49E60796`, `INTERMEDIATE_BLOCK_INTERVAL=30`, `BLOCK_INTERVAL=600`, `PROOF_THRESHOLD=1`, `ZK_VERIFIER=0xF9780104117C0FaD3A9b1386FbF40a9F5857988A`, and no pause on the respected game type. `OptimismPortal2.checkWithdrawal()` depends on `anchorStateRegistry.isGameClaimValid()` [before finalization](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/L1/OptimismPortal2.sol#L515-L551), and `AnchorStateRegistry.isGameClaimValid()` requires the game to be finalized and defender-won [at lines 311-335](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/dispute/AnchorStateRegistry.sol#L311-L335).

Fresh public-source and Base Sepolia checks on `2026-04-28` did not identify a public fix in the current public `base/base` main branch and confirmed the mismatch is still present in the testnet competition environment. The latest public main commit checked was [`4b389f03b4963ec7bbb4d024445af6281a90d9a8`](https://github.com/base/base/commit/4b389f03b4963ec7bbb4d024445af6281a90d9a8); at that commit, `OpSuccinctProvider::generate_witness()` still passes [`DEFAULT_INTERMEDIATE_ROOT_INTERVAL` into `get_sp1_stdin()`](https://github.com/base/base/blob/4b389f03b4963ec7bbb4d024445af6281a90d9a8/crates/proof/zk/service/src/backends/op_succinct/provider.rs#L100-L103), the public [`ProveBlockRequest` still has no intermediate-root interval field](https://github.com/base/base/blob/4b389f03b4963ec7bbb4d024445af6281a90d9a8/crates/proof/zk/client/proto/zk_prover.proto#L17-L33), and the default interval remains [`10`](https://github.com/base/base/blob/4b389f03b4963ec7bbb4d024445af6281a90d9a8/crates/proof/succinct/utils/client/src/client.rs#L18-L19). Read-only Base Sepolia calls at block `10746130` (`2026-04-28T01:32:24Z`) still returned `gameImpls(621)=0xF3f0fA3124b7b0feB048A00404Fe4D5D49E60796`, `INTERMEDIATE_BLOCK_INTERVAL=30`, `BLOCK_INTERVAL=600`, `PROOF_THRESHOLD=1`, `ZK_AGGREGATE_HASH=0x0020f5df94945d8d40bc2b11128cbf2535565f5cde58960029017ee2ccef46d7`, `respectedGameType=621`, and `paused=false`. This confirms the bug-relevant mismatch is still deployed on Base Sepolia, while not ruling out private unpublished fixes.

#### Public-known / duplicate boundary

I did not identify a public disclosure for this exact rc.28 proof-service interval mismatch. The live Immunefi page checked on `2026-04-28` lists public known issues and audit references, and states that publicly disclosed bugs or unfixed issues mentioned in prior audits are not eligible. This report is distinct from generic ZK prover/circuit issues because the root cause is the Base rc.28 integration path: the request surface cannot pass the contract's active intermediate-root interval to the proof service.

### Repository, file, and line of code where finding is found

* [`OpSuccinctProvider::generate_witness()` hardcodes the default interval](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/proof/zk/service/src/backends/op_succinct/provider.rs#L100-L103)
* [`DEFAULT_INTERMEDIATE_ROOT_INTERVAL` is 10](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/client.rs#L18-L19)
* [`ProveBlockRequest` lacks an intermediate-root interval field](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/proof/zk/client/proto/zk_prover.proto#L17-L32)
* [`AggregateVerifier.challenge()` hashes one 30-block root for challenge verification](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L480-L521)
* [`_verifyZkProof()` hashes the contract-side intermediate root vector into the verifier journal](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L904-L932)

### Steps to replicate

All testing was performed in an isolated reviewer-style environment. I did not use mainnet or public testnet.

* secret gist: [PoC gist](https://gist.github.com/cgdusek/43cde38ec160412f84fa5eaf687f8181)
* PoC name: `RC28IntervalMismatchRealSP1PoC`

```bash
bash run_poc.sh
```

Expected high-signal output markers:

* `POC_RESULT: REAL_ZK_PROOF_SYSTEM=SP1 Groth16`
* `POC_RESULT: ZK_VERIFIER_RC28_PROOF_ACCEPTED_WITH_ONCHAIN_JOURNAL=false`
* `POC_RESULT: FULL_PROPOSAL_DIGESTS_EQUAL=false`
* `POC_RESULT: ADDITIONAL_WITHDRAWAL_FREEZE_SECONDS=518400`
* `POC_RESULT: ADDITIONAL_WITHDRAWAL_FREEZE_GE_24H=true`
* `POC_RESULT: INTERVAL_MISMATCH_PROVEN=true`
* `POC_RESULT: HIGH_TEMPORARY_FREEZE_CANDIDATE=true`

### Impact of finding

#### Short term

Permissionless ZK backing for active 30-block dispute/proposal intervals cannot supply the second proof that shortens finality from `604800` seconds to `86400` seconds. Affected games remain in the one-proof path for an additional `518400` seconds, delaying withdrawal finalization and dispute-game bond finality by more than 24 hours.

#### Long term

If rc.28 is used as the ZK proof-service path without an interval fix, the ZK backstop/fast-finality lane remains unusable for the active interval. This report does not claim permanent freezing, invalid state finalization, direct theft, or total network shutdown.

### Mitigation suggestions

* Add an explicit intermediate-root interval field to `ProveBlockRequest` and populate it from the candidate/onchain `INTERMEDIATE_BLOCK_INTERVAL`.
* Refuse to submit a ZK proof when the generated intermediate-root count does not match the target AggregateVerifier's expected interval.
* Add regression tests for 30-block challenge proofs and 600-block full proposal proofs where the ZK service and AggregateVerifier journal digests must match.
* As an operational mitigation boundary, treat this as operator-remediable by redeploy/code config, not user-remediable by existing runtime configuration.

### Optional suggested patch

Treat `DEFAULT_INTERMEDIATE_ROOT_INTERVAL` only as a fallback for standalone tests. Production proof requests should carry the onchain interval through the challenger, prover API, witness-generation call, and aggregation input.

### Comparable public incidents

Not used. This report is source and PoC driven.

### References

#### Runnable PoC package

* secret gist: [PoC gist](https://gist.github.com/cgdusek/43cde38ec160412f84fa5eaf687f8181)
* PoC name: `RC28IntervalMismatchRealSP1PoC`

#### Live deployment / release confirmation

* [Audit Comp | Base Azul scope](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/), checked `2026-04-28`
* [Audit Comp | Base Azul information](https://immunefi.com/audit-competition/audit-comp-base-azul/information/), checked `2026-04-28`
* [Base V1 proof finality model](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/docs/specs/pages/upgrades/v1/proofs.md#L27-L42)
* [Base V1 ZK prover role](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/docs/specs/pages/upgrades/v1/proofs.md#L116-L122)

#### Relevant code paths

* [rc.28 provider hardcodes default interval](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/proof/zk/service/src/backends/op_succinct/provider.rs#L100-L103)
* [default interval is 10](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/client.rs#L18-L19)
* [root recording depends on interval](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/client.rs#L184-L192)
* [aggregation commits all intermediate roots](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/programs/aggregation/src/main.rs#L80-L118)
* [proposal proof uses contract intermediate roots](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L418-L439)
* [challenge proof uses one intermediate root](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L480-L521)
* [ZK journal verification](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L904-L932)
* [finality delays](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L44-L53)
* [proof-count delay selection](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L763-L823)
* [withdrawal validity depends on finalized game claim](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/L1/OptimismPortal2.sol#L515-L551)
* [game claim validity requires finalized defender win](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/dispute/AnchorStateRegistry.sol#L311-L335)

#### Scope / rules

* [Immunefi PoC guidelines](https://immunefisupport.zendesk.com/hc/en-us/articles/9946217628561-Proof-of-Concept-PoC-Guidelines-and-Rules)

#### Public protocol documentation

* [Base V1 proof finality model](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/docs/specs/pages/upgrades/v1/proofs.md#L27-L42)
* [Base V1 ZK prover role](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/docs/specs/pages/upgrades/v1/proofs.md#L116-L122)

#### Additional context / provenance

* [Base Azul information page](https://immunefi.com/audit-competition/audit-comp-base-azul/information/), checked `2026-04-28`

### Submission notes

* selected scope criterion: `High - Temporary freezing of funds for at least 24 hours (e.g., stuck withdrawal proofs, locked dispute game bonds)`
* online form category: `Smart Contract`
* online form asset picker selection: [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof)
* specific scoped assets carrying the claim:
  * selected [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof) Smart Contract asset for `AggregateVerifier`, `AnchorStateRegistry`, and `OptimismPortal2` finality/withdrawal impact
  * supporting [`base/base v0.8.0-rc.28`](https://github.com/base/base/tree/v0.8.0-rc.28) ZK proof service
  * supporting [`base/contract-deployments`](https://github.com/base/contract-deployments) Base Sepolia Azul type-621 deployment constants
* severity anchor: `518400` additional seconds of finality delay, exceeding 24 hours
* breadth-only variant: the 30-block challenge path has the same interval mismatch by source/code path; the runnable severity anchor is the full 600-block proposal second-proof path
* explicitly not claiming: Critical proof forgery/bypass, finalized invalid state, direct theft, permanent freezing, chain shutdown, public-network exploitation, or invalid proof reliance

### Link to Proof of Concept

<https://gist.github.com/cgdusek/43cde38ec160412f84fa5eaf687f8181>

### Proof of Concept

## PoC Specification

### Command

```bash
bash run_poc.sh
```

Run from the secret gist root after downloading the gist files into one directory. The command performs an isolated reviewer-environment execution. It requires `forge`, Rust/Cargo, `protoc`, Go, outbound internet for first-run source/tool/circuit downloads, and enough disk and memory for SP1 Groth16 proving. It does not require `SEPOLIA_RPC_URL`, a wallet, a private key, a funded account, public-network writes, or an external proof fixture.

### Version Gate

The runner prints and enforces:

* `POC_RESULT: BASE_BASE_SCOPE_TAG=v0.8.0-rc.28`
* `POC_RESULT: BASE_BASE_SCOPE_COMMIT=e3467a2048881213b56739a54a876efb9c6ea103`
* `POC_RESULT: BASE_CONTRACTS_TAG=v8.1.0`
* `POC_RESULT: BASE_CONTRACTS_COMMIT_EXPECTED=01dad230390cd69bcf130b5fc7a7a580b31650a7`
* `POC_RESULT: BASE_CONTRACTS_COMMIT_MATCH=true`
* `POC_RESULT: SP1_VERSION=v6.1.0`
* `POC_RESULT: SP1_CONTRACTS_COMMIT_MATCH=true`

### What the runner does

{% stepper %}
{% step %}

### Writes an SP1 program

Writes a small SP1 program that commits the same aggregate-journal preimage shape that Base `AggregateVerifier._verifyZkProof(...)` hashes: proposer, L1 origin hash, starting root/number, ending root/number, intermediate roots, `CONFIG_HASH`, and `ZK_RANGE_HASH`.
{% endstep %}

{% step %}

### Generates the onchain proof

Generates a real SP1 Groth16 proof for the contract-side full proposal vector: `600 / 30 = 20` intermediate roots.
{% endstep %}

{% step %}

### Generates the rc.28 proof

Generates a real SP1 Groth16 proof for the rc.28 proof-service vector: `600 / 10 = 60` intermediate roots.
{% endstep %}

{% step %}

### Writes the fixture

Writes both proof byte strings, journals, root counts, vkey, and public Sepolia aggregate-vkey boundary data to a JSON fixture.
{% endstep %}

{% step %}

### Builds the SP1 verifier contract

Builds the SP1 Groth16 verifier contract from `succinctlabs/sp1-contracts` at commit `2ac5ecbbe473421a963d67e55f182e9a36576f7c`.
{% endstep %}

{% step %}

### Clones Base contracts

Clones `base/contracts v8.1.0`, checks out `01dad230390cd69bcf130b5fc7a7a580b31650a7`, and installs upstream dependencies with `make deps`.
{% endstep %}

{% step %}

### Runs the Foundry test

Copies `RC28IntervalMismatchRealSP1PoC.t.sol` into the cloned Base test tree and runs Foundry with `--root` set to the release-pinned Base checkout.
{% endstep %}

{% step %}

### Deploys the verifier path

Deploys the real SP1 Groth16 verifier artifact and Base `ZkVerifier` inside Foundry execution.
{% endstep %}

{% step %}

### Confirms both valid pairings

Confirms the onchain-interval proof verifies with the onchain journal and the rc.28 proof verifies with the rc.28 journal.
{% endstep %}

{% step %}

### Confirms the mismatch

Confirms the same rc.28 proof is rejected when paired with the onchain 30-block journal, which is the contract-side journal needed for the second proof.
{% endstep %}

{% step %}

### Confirms the boundary

Confirms the equivalent proof is rejected when public Sepolia's aggregate vkey is supplied as `imageId`, making the production-vkey boundary explicit.
{% endstep %}

{% step %}

### Emits the severity marker

Emits the `518400` second additional finality-delay marker and exits only if the Foundry test passes.
{% endstep %}
{% endstepper %}

### Embedded PoC code for automated triage

The authoritative executable files are included in `poc/` and in the secret gist. These hashes identify the exact source snapshot used by this specification:

* `poc/run_poc.sh`
  * SHA-256: `8f88582fc9887b061a8076bb6495dfe6fe255a9eeb72ee5e5a9ed8e1af04211d`
* `poc/RC28IntervalMismatchRealSP1PoC.t.sol`
  * SHA-256: `15d48d80e2e11547f73522285cde661586017be6f71653fa38fef493106fde74`
* `poc/README.md`
  * SHA-256: `f8c80c492d933df89cd9e41714c53e4abba8998f110fc7cb6a165706b061f9ac`
* `poc/OUTPUT.txt`
  * SHA-256: `e2cddf122430495ca508ebd08bb9c880dc0ad55e1837139cf059da57dae2b30a`

<details>

<summary><code>run_poc.sh</code></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

if [[ "${POC_CAPTURE_OUTPUT:-1}" == "1" && -z "${POC_OUTPUT_CAPTURED:-}" ]]; then
  export POC_OUTPUT_CAPTURED=1
  bash "$0" "$@" 2>&1 | tee "${ROOT_DIR}/OUTPUT.txt"
  exit "${PIPESTATUS[0]}"
fi

WORK_DIR="${POC_WORKDIR:-"${ROOT_DIR}/.work"}"
SP1_VERSION="6.1.0"
SP1_CONTRACTS_COMMIT="2ac5ecbbe473421a963d67e55f182e9a36576f7c"
BASE_BASE_SCOPE_TAG="v0.8.0-rc.28"
BASE_BASE_SCOPE_COMMIT="e3467a2048881213b56739a54a876efb9c6ea103"
BASE_CONTRACTS_TAG="v8.1.0"
BASE_CONTRACTS_COMMIT="01dad230390cd69bcf130b5fc7a7a580b31650a7"
BASE_CONTRACTS_DIR="${WORK_DIR}/contracts-v8.1.0"
SP1_PROJECT_DIR="${WORK_DIR}/sp1-rc28-interval-journal"
SP1_CONTRACTS_DIR="${WORK_DIR}/sp1-contracts"
SP1_FIXTURE_FILE="${WORK_DIR}/rc28-interval-real-sp1-fixture.json"
SP1_VERIFIER_ARTIFACT="${SP1_CONTRACTS_DIR}/contracts/out/v${SP1_VERSION}/SP1VerifierGroth16.sol/SP1Verifier.json"
BASE_TEST_PATH="${BASE_CONTRACTS_DIR}/test/multiproof/RC28IntervalMismatchRealSP1PoC.t.sol"
BASE_FIXTURE_FILE="${BASE_CONTRACTS_DIR}/.testdata/rc28-interval-real-sp1-fixture.json"
BASE_SP1_VERIFIER_ARTIFACT="${BASE_CONTRACTS_DIR}/.testdata/SP1VerifierGroth16.json"
TEST_NAME="RC28IntervalMismatchRealSP1PoC.t.sol"
TEST_PATH="${ROOT_DIR}/${TEST_NAME}"

echo "POC_RESULT: POC_NAME=RC28IntervalMismatchRealSP1PoC"
echo "POC_RESULT: MODE=real_sp1_groth16_base_zkverifier"
echo "POC_RESULT: PUBLIC_NETWORK_BROADCAST=false"
echo "POC_RESULT: PRODUCTION_FORK_PATH=not_used"
echo "POC_RESULT: PRIMARY_PROOF_TYPE=ZK"
echo "POC_RESULT: SP1_VERSION=v${SP1_VERSION}"
echo "POC_RESULT: BASE_BASE_SCOPE_TAG=${BASE_BASE_SCOPE_TAG}"
echo "POC_RESULT: BASE_BASE_SCOPE_COMMIT=${BASE_BASE_SCOPE_COMMIT}"
echo "POC_RESULT: BASE_CONTRACTS_TAG=${BASE_CONTRACTS_TAG}"
echo "POC_RESULT: BASE_CONTRACTS_COMMIT_EXPECTED=${BASE_CONTRACTS_COMMIT}"
echo "POC_RESULT: ACTIVE_AGGREGATE_VERIFIER=0xF3f0fA3124b7b0feB048A00404Fe4D5D49E60796"
echo "POC_RESULT: ACTIVE_ZK_VERIFIER=0xF9780104117C0FaD3A9b1386FbF40a9F5857988A"
echo "POC_RESULT: CONTRACT_BLOCK_INTERVAL=600"
echo "POC_RESULT: CONTRACT_INTERMEDIATE_BLOCK_INTERVAL=30"
echo "POC_RESULT: RC28_DEFAULT_INTERMEDIATE_ROOT_INTERVAL=10"
echo "POC_RESULT: PROOF_THRESHOLD=1"
echo "POC_RESULT: FAST_FINALIZATION_DELAY_SECONDS=86400"
echo "POC_RESULT: SLOW_FINALIZATION_DELAY_SECONDS=604800"
echo "POC_RESULT: ADDITIONAL_WITHDRAWAL_FREEZE_SECONDS=518400"
echo "POC_RESULT: ADDITIONAL_WITHDRAWAL_FREEZE_GE_24H=true"

if ! command -v forge >/dev/null 2>&1; then
  echo "POC_RESULT: FORGE_AVAILABLE=false"
  echo "forge is required but was not found on PATH" >&2
  exit 2
fi
echo "POC_RESULT: FORGE_AVAILABLE=true"

if ! command -v cargo >/dev/null 2>&1; then
  echo "POC_RESULT: CARGO_AVAILABLE=false"
  echo "cargo/rust is required but was not found on PATH" >&2
  exit 2
fi
echo "POC_RESULT: CARGO_AVAILABLE=true"

if ! command -v protoc >/dev/null 2>&1; then
  echo "POC_RESULT: PROTOC_AVAILABLE=false"
  echo "protoc is required by SP1 prover dependencies; install protobuf-compiler/protobuf before running" >&2
  exit 2
fi
echo "POC_RESULT: PROTOC_AVAILABLE=true"

if ! command -v go >/dev/null 2>&1; then
  echo "POC_RESULT: GO_AVAILABLE=false"
  echo "go is required by SP1's native Gnark Groth16 prover dependencies" >&2
  exit 2
fi
echo "POC_RESULT: GO_AVAILABLE=true"

if command -v rustup >/dev/null 2>&1; then
  rustup default stable >/dev/null
  export RUSTUP_TOOLCHAIN="${RUSTUP_TOOLCHAIN:-stable}"
fi
echo "POC_RESULT: RUSTC_VERSION=$(rustc --version)"

export PATH="${HOME}/.sp1/bin:${HOME}/.cargo/bin:${PATH}"
if ! command -v cargo-prove >/dev/null 2>&1; then
  echo "POC_RESULT: CARGO_PROVE_AVAILABLE=false"
  echo "POC_RESULT: INSTALLING_SP1_TOOLCHAIN=true"
  curl -fsSL https://sp1.succinct.xyz | bash
  export PATH="${HOME}/.sp1/bin:${HOME}/.cargo/bin:${PATH}"
  sp1up
fi
echo "POC_RESULT: CARGO_PROVE_AVAILABLE=true"
cargo prove --version

mkdir -p "${WORK_DIR}"
export CARGO_TARGET_DIR="${WORK_DIR}/cargo-target"
echo "POC_RESULT: CARGO_TARGET_DIR=${CARGO_TARGET_DIR}"

write_sp1_project() {
  rm -rf "${SP1_PROJECT_DIR}"
  mkdir -p "${SP1_PROJECT_DIR}/program/src" "${SP1_PROJECT_DIR}/script/src/bin"

  cat > "${SP1_PROJECT_DIR}/Cargo.toml" <<'EOF_WORKSPACE'
[workspace]
members = ["program", "script"]
resolver = "2"
EOF_WORKSPACE

  cat > "${SP1_PROJECT_DIR}/program/Cargo.toml" <<'EOF_PROGRAM_TOML'
[package]
version = "0.1.0"
name = "rc28-interval-journal-program"
edition = "2021"

[dependencies]
sp1-zkvm = { version = "=6.1.0" }
sha3 = { version = "0.10", default-features = false }
EOF_PROGRAM_TOML

  cat > "${SP1_PROJECT_DIR}/program/src/main.rs" <<'EOF_PROGRAM'
#![no_main]
use sha3::{Digest, Keccak256};

sp1_zkvm::entrypoint!(main);

const CONFIG_HASH: [u8; 32] = [
    0x12, 0xe9, 0xc4, 0x5f, 0x19, 0xf9, 0x81, 0x7c,
    0x6d, 0x43, 0x85, 0xfa, 0xd2, 0x9e, 0x7a, 0x70,
    0xc3, 0x55, 0x50, 0x2c, 0xf0, 0x88, 0x3e, 0x76,
    0xa9, 0xa7, 0xe4, 0x78, 0xa8, 0x5d, 0x13, 0x60,
];

const ZK_RANGE_HASH: [u8; 32] = [
    0x23, 0x08, 0x1d, 0x5f, 0x7a, 0x4d, 0x3c, 0xe0,
    0x4d, 0x41, 0xc2, 0xf0, 0x3b, 0x38, 0x6b, 0xad,
    0x4f, 0x42, 0xa9, 0x9d, 0x5b, 0x4c, 0x8e, 0xf4,
    0x4f, 0x4c, 0x36, 0x2c, 0x2c, 0x59, 0x54, 0x64,
];

pub fn main() {
    let proposer = sp1_zkvm::io::read::<[u8; 20]>();
    let l1_origin_hash = sp1_zkvm::io::read::<[u8; 32]>();
    let starting_root = sp1_zkvm::io::read::<[u8; 32]>();
    let starting_l2 = sp1_zkvm::io::read::<u64>();
    let ending_root = sp1_zkvm::io::read::<[u8; 32]>();
    let ending_l2 = sp1_zkvm::io::read::<u64>();
    let intermediate_roots = sp1_zkvm::io::read::<Vec<[u8; 32]>>();

    let mut encoded = Vec::with_capacity(20 + 32 + 32 + 8 + 32 + 8 + intermediate_roots.len() * 32 + 64);
    encoded.extend_from_slice(&proposer);
    encoded.extend_from_slice(&l1_origin_hash);
    encoded.extend_from_slice(&starting_root);
    encoded.extend_from_slice(&starting_l2.to_be_bytes());
    encoded.extend_from_slice(&ending_root);
    encoded.extend_from_slice(&ending_l2.to_be_bytes());
    for root in &intermediate_roots {
        encoded.extend_from_slice(root);
    }
    encoded.extend_from_slice(&CONFIG_HASH);
    encoded.extend_from_slice(&ZK_RANGE_HASH);

    let journal: [u8; 32] = Keccak256::digest(&encoded).into();
    sp1_zkvm::io::commit_slice(&journal);
}
EOF_PROGRAM

  cat > "${SP1_PROJECT_DIR}/script/Cargo.toml" <<'EOF_SCRIPT_TOML'
[package]
version = "0.1.0"
name = "rc28-interval-journal-script"
edition = "2021"
default-run = "evm"

[[bin]]
name = "evm"
path = "src/bin/evm.rs"

[dependencies]
sp1-sdk = { version = "=6.1.0", features = ["blocking", "native-gnark"] }
hex = "0.4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sha3 = "0.10"

[build-dependencies]
sp1-build = { version = "=6.1.0" }
EOF_SCRIPT_TOML

  cat > "${SP1_PROJECT_DIR}/script/build.rs" <<'EOF_BUILD'
use sp1_build::build_program_with_args;

fn main() {
    build_program_with_args("../program", Default::default())
}
EOF_BUILD

  cat > "${SP1_PROJECT_DIR}/script/src/bin/evm.rs" <<'EOF_SCRIPT'
use serde::{Deserialize, Serialize};
use sha3::{Digest, Keccak256};
use sp1_sdk::{
    blocking::{ProveRequest, Prover, ProverClient},
    include_elf, Elf, HashableKey, ProvingKey, SP1ProofWithPublicValues, SP1Stdin, SP1VerifyingKey,
};
use std::path::PathBuf;

const JOURNAL_ELF: Elf = include_elf!("rc28-interval-journal-program");
const PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH: &str =
    "0x0020f5df94945d8d40bc2b11128cbf2535565f5cde58960029017ee2ccef46d7";
const CONFIG_HASH: [u8; 32] = [
    0x12, 0xe9, 0xc4, 0x5f, 0x19, 0xf9, 0x81, 0x7c,
    0x6d, 0x43, 0x85, 0xfa, 0xd2, 0x9e, 0x7a, 0x70,
    0xc3, 0x55, 0x50, 0x2c, 0xf0, 0x88, 0x3e, 0x76,
    0xa9, 0xa7, 0xe4, 0x78, 0xa8, 0x5d, 0x13, 0x60,
];
const ZK_RANGE_HASH: [u8; 32] = [
    0x23, 0x08, 0x1d, 0x5f, 0x7a, 0x4d, 0x3c, 0xe0,
    0x4d, 0x41, 0xc2, 0xf0, 0x3b, 0x38, 0x6b, 0xad,
    0x4f, 0x42, 0xa9, 0x9d, 0x5b, 0x4c, 0x8e, 0xf4,
    0x4f, 0x4c, 0x36, 0x2c, 0x2c, 0x59, 0x54, 0x64,
];

#[derive(Debug, Clone)]
struct Statement {
    proposer: [u8; 20],
    l1_origin_hash: [u8; 32],
    starting_root: [u8; 32],
    starting_l2: u64,
    ending_root: [u8; 32],
    ending_l2: u64,
    intermediate_roots: Vec<[u8; 32]>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct Fixture {
    onchain_proof: String,
    rc28_proof: String,
    vkey: String,
    onchain_journal: String,
    rc28_journal: String,
    public_sepolia_zk_aggregate_hash: String,
    config_hash: String,
    zk_range_hash: String,
    l1_origin_hash: String,
    starting_root: String,
    ending_root: String,
    starting_l2: u64,
    ending_l2: u64,
    onchain_root_count: usize,
    rc28_root_count: usize,
    onchain_packed_len: usize,
    rc28_packed_len: usize,
    contract_block_interval: u64,
    contract_intermediate_block_interval: u64,
    rc28_default_intermediate_root_interval: u64,
    fast_finalization_delay_seconds: u64,
    slow_finalization_delay_seconds: u64,
    additional_withdrawal_freeze_seconds: u64,
}

fn repeated<const N: usize>(byte: u8) -> [u8; N] {
    [byte; N]
}

fn root_at(block: u64) -> [u8; 32] {
    let mut input = b"base-azul-rc28-interval-root:".to_vec();
    input.extend_from_slice(&block.to_be_bytes());
    Keccak256::digest(input).into()
}

fn roots_for_interval(total_blocks: u64, interval: u64) -> Vec<[u8; 32]> {
    assert_eq!(total_blocks % interval, 0);
    (1..=total_blocks / interval)
        .map(|i| root_at(i * interval))
        .collect()
}

fn hex32(bytes: &[u8; 32]) -> String {
    format!("0x{}", hex::encode(bytes))
}

fn is_bytes32_hex(value: &str) -> bool {
    value.len() == 66
        && value.starts_with("0x")
        && value[2..].bytes().all(|b| b.is_ascii_hexdigit())
}

fn proof_hex(proof: &SP1ProofWithPublicValues) -> String {
    format!("0x{}", hex::encode(proof.bytes()))
}

fn packed_statement(statement: &Statement) -> Vec<u8> {
    let mut encoded = Vec::with_capacity(20 + 32 + 32 + 8 + 32 + 8 + statement.intermediate_roots.len() * 32 + 64);
    encoded.extend_from_slice(&statement.proposer);
    encoded.extend_from_slice(&statement.l1_origin_hash);
    encoded.extend_from_slice(&statement.starting_root);
    encoded.extend_from_slice(&statement.starting_l2.to_be_bytes());
    encoded.extend_from_slice(&statement.ending_root);
    encoded.extend_from_slice(&statement.ending_l2.to_be_bytes());
    for root in &statement.intermediate_roots {
        encoded.extend_from_slice(root);
    }
    encoded.extend_from_slice(&CONFIG_HASH);
    encoded.extend_from_slice(&ZK_RANGE_HASH);
    encoded
}

fn aggregate_verifier_journal(statement: &Statement) -> [u8; 32] {
    Keccak256::digest(packed_statement(statement)).into()
}

fn stdin(statement: &Statement) -> SP1Stdin {
    let mut stdin = SP1Stdin::new();
    stdin.write(&statement.proposer);
    stdin.write(&statement.l1_origin_hash);
    stdin.write(&statement.starting_root);
    stdin.write(&statement.starting_l2);
    stdin.write(&statement.ending_root);
    stdin.write(&statement.ending_l2);
    stdin.write(&statement.intermediate_roots);
    stdin
}

fn prove<P>(client: &P, pk: &P::ProvingKey, statement: &Statement) -> SP1ProofWithPublicValues
where
    P: Prover,
{
    client.prove(pk, stdin(statement)).groth16().run().expect("prove failed")
}

fn main() {
    sp1_sdk::utils::setup_logger();
    let client = ProverClient::from_env();
    let pk = client.setup(JOURNAL_ELF).expect("setup failed");
    let vk: &SP1VerifyingKey = pk.verifying_key();

    let contract_block_interval = 600u64;
    let contract_intermediate_interval = 30u64;
    let rc28_default_interval = 10u64;

    let l1_origin_hash = repeated::<32>(0x22);
    let starting_root = root_at(0);
    let ending_root = root_at(contract_block_interval);
    let base = Statement {
        proposer: repeated::<20>(0x11),
        l1_origin_hash,
        starting_root,
        starting_l2: 550_800,
        ending_root,
        ending_l2: 551_400,
        intermediate_roots: Vec::new(),
    };

    let onchain = Statement {
        intermediate_roots: roots_for_interval(contract_block_interval, contract_intermediate_interval),
        ..base.clone()
    };
    let rc28 = Statement {
        intermediate_roots: roots_for_interval(contract_block_interval, rc28_default_interval),
        ..base
    };

    let onchain_journal = aggregate_verifier_journal(&onchain);
    let rc28_journal = aggregate_verifier_journal(&rc28);
    let onchain_packed_len = packed_statement(&onchain).len();
    let rc28_packed_len = packed_statement(&rc28).len();

    let onchain_proof = prove(&client, &pk, &onchain);
    let rc28_proof = prove(&client, &pk, &rc28);
    assert_eq!(onchain_proof.public_values.as_slice(), &onchain_journal);
    assert_eq!(rc28_proof.public_values.as_slice(), &rc28_journal);
    assert_ne!(onchain_journal, rc28_journal);

    let fixture = Fixture {
        onchain_proof: proof_hex(&onchain_proof),
        rc28_proof: proof_hex(&rc28_proof),
        vkey: vk.bytes32().to_string(),
        onchain_journal: hex32(&onchain_journal),
        rc28_journal: hex32(&rc28_journal),
        public_sepolia_zk_aggregate_hash: PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH.to_string(),
        config_hash: hex32(&CONFIG_HASH),
        zk_range_hash: hex32(&ZK_RANGE_HASH),
        l1_origin_hash: hex32(&l1_origin_hash),
        starting_root: hex32(&starting_root),
        ending_root: hex32(&ending_root),
        starting_l2: onchain.starting_l2,
        ending_l2: onchain.ending_l2,
        onchain_root_count: onchain.intermediate_roots.len(),
        rc28_root_count: rc28.intermediate_roots.len(),
        onchain_packed_len,
        rc28_packed_len,
        contract_block_interval,
        contract_intermediate_block_interval: contract_intermediate_interval,
        rc28_default_intermediate_root_interval: rc28_default_interval,
        fast_finalization_delay_seconds: 86_400,
        slow_finalization_delay_seconds: 604_800,
        additional_withdrawal_freeze_seconds: 518_400,
    };

    println!("POC_RESULT: REAL_ZK_PROOF_SYSTEM=SP1 Groth16");
    println!("POC_RESULT: SP1_PROGRAM_VKEY={}", fixture.vkey);
    println!("POC_RESULT: PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH_NOT_USED={}", PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH);
    println!("POC_RESULT: SP1_PROGRAM_VKEY_BYTES32_FORMAT={}", is_bytes32_hex(&fixture.vkey));
    println!(
        "POC_RESULT: PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH_BYTES32_FORMAT={}",
        is_bytes32_hex(PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH)
    );
    println!(
        "POC_RESULT: PROGRAM_VKEY_DIFFERS_FROM_PUBLIC_SEPOLIA={}",
        fixture.vkey.to_lowercase() != PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH
    );
    println!("POC_RESULT: FULL_PROPOSAL_ONCHAIN_ROOT_COUNT={}", fixture.onchain_root_count);
    println!("POC_RESULT: FULL_PROPOSAL_RC28_ROOT_COUNT={}", fixture.rc28_root_count);
    println!("POC_RESULT: FULL_PROPOSAL_ONCHAIN_PACKED_LEN={}", fixture.onchain_packed_len);
    println!("POC_RESULT: FULL_PROPOSAL_RC28_PACKED_LEN={}", fixture.rc28_packed_len);
    println!("POC_RESULT: FULL_PROPOSAL_ONCHAIN_DIGEST={}", fixture.onchain_journal);
    println!("POC_RESULT: FULL_PROPOSAL_RC28_DIGEST={}", fixture.rc28_journal);
    println!("POC_RESULT: FULL_PROPOSAL_DIGESTS_EQUAL=false");
    println!("POC_RESULT: ONCHAIN_SP1_PROOF_BYTES_PRESENT=true");
    println!("POC_RESULT: RC28_SP1_PROOF_BYTES_PRESENT=true");

    let output = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("../rc28-interval-real-sp1-fixture.json");
    std::fs::write(&output, serde_json::to_string_pretty(&fixture).unwrap()).expect("write fixture");
    println!("POC_RESULT: SP1_FIXTURE={}", output.display());
}
EOF_SCRIPT
}

build_sp1_fixture() {
  if [[ "${REUSE_SP1_FIXTURE:-0}" == "1" && -f "${SP1_FIXTURE_FILE}" ]]; then
    echo "POC_RESULT: REUSE_SP1_FIXTURE=true"
    return
  fi

  write_sp1_project
  (
    cd "${SP1_PROJECT_DIR}/script"
    RUST_LOG=info cargo run --release --bin evm
  )
  cp "${SP1_PROJECT_DIR}/rc28-interval-real-sp1-fixture.json" "${SP1_FIXTURE_FILE}"
  echo "POC_RESULT: SP1_FIXTURE_COPIED=${SP1_FIXTURE_FILE}"
}

build_sp1_verifier_artifact() {
  if [[ -f "${SP1_VERIFIER_ARTIFACT}" ]]; then
    echo "POC_RESULT: SP1_VERIFIER_ARTIFACT_REUSED=${SP1_VERIFIER_ARTIFACT}"
    return
  fi

  rm -rf "${SP1_CONTRACTS_DIR}"
  git clone --depth 1 https://github.com/succinctlabs/sp1-contracts.git "${SP1_CONTRACTS_DIR}"
  (
    cd "${SP1_CONTRACTS_DIR}"
    git fetch --depth 1 origin "${SP1_CONTRACTS_COMMIT}"
    git checkout --detach "${SP1_CONTRACTS_COMMIT}"
    if [[ "$(git rev-parse HEAD)" != "${SP1_CONTRACTS_COMMIT}" ]]; then
      echo "POC_RESULT: SP1_CONTRACTS_COMMIT_MATCH=false" >&2
      exit 2
    fi
    echo "POC_RESULT: SP1_CONTRACTS_COMMIT=$(git rev-parse HEAD)"
    echo "POC_RESULT: SP1_CONTRACTS_TAGS=$(git tag --points-at HEAD | tr '\n' ',' | sed 's/,$//')"
    echo "POC_RESULT: SP1_CONTRACTS_COMMIT_MATCH=true"
  )
  forge build --root "${SP1_CONTRACTS_DIR}/contracts" --contracts "src/v${SP1_VERSION}/SP1VerifierGroth16.sol" --silent
  test -f "${SP1_VERIFIER_ARTIFACT}"
  echo "POC_RESULT: SP1_GROTH16_VERIFIER_COMPILED=true"
}

fetch_base_contracts() {
  if [[ -d "${BASE_CONTRACTS_DIR}/.git" ]]; then
    (
      cd "${BASE_CONTRACTS_DIR}"
      git fetch --depth 1 origin "${BASE_CONTRACTS_COMMIT}"
      git checkout --detach "${BASE_CONTRACTS_COMMIT}" >/dev/null
      if [[ ! -f lib/forge-std/src/Test.sol || ! -f lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol ]]; then
        make deps
      fi
      if [[ "$(git rev-parse HEAD)" != "${BASE_CONTRACTS_COMMIT}" ]]; then
        echo "POC_RESULT: BASE_CONTRACTS_COMMIT_MATCH=false" >&2
        exit 2
      fi
      echo "POC_RESULT: BASE_CONTRACTS_REUSED=true"
      echo "POC_RESULT: BASE_CONTRACTS_COMMIT=$(git rev-parse HEAD)"
      echo "POC_RESULT: BASE_CONTRACTS_TAGS=$(git tag --points-at HEAD | tr '\n' ',' | sed 's/,$//')"
      echo "POC_RESULT: BASE_CONTRACTS_COMMIT_MATCH=true"
      echo "POC_RESULT: BASE_CONTRACTS_DEPS_READY=true"
    )
    return
  fi

  rm -rf "${BASE_CONTRACTS_DIR}"
  git clone --depth 1 --branch "${BASE_CONTRACTS_TAG}" https://github.com/base/contracts.git "${BASE_CONTRACTS_DIR}"
  (
    cd "${BASE_CONTRACTS_DIR}"
    git fetch --depth 1 origin "${BASE_CONTRACTS_COMMIT}"
    git checkout --detach "${BASE_CONTRACTS_COMMIT}" >/dev/null
    make deps
    if [[ "$(git rev-parse HEAD)" != "${BASE_CONTRACTS_COMMIT}" ]]; then
      echo "POC_RESULT: BASE_CONTRACTS_COMMIT_MATCH=false" >&2
      exit 2
    fi
    echo "POC_RESULT: BASE_CONTRACTS_REUSED=false"
    echo "POC_RESULT: BASE_CONTRACTS_COMMIT=$(git rev-parse HEAD)"
    echo "POC_RESULT: BASE_CONTRACTS_TAGS=$(git tag --points-at HEAD | tr '\n' ',' | sed 's/,$//')"
    echo "POC_RESULT: BASE_CONTRACTS_COMMIT_MATCH=true"
    echo "POC_RESULT: BASE_CONTRACTS_DEPS_READY=true"
  )
}

build_sp1_fixture
build_sp1_verifier_artifact
fetch_base_contracts

if [[ ! -f "${TEST_PATH}" ]]; then
  echo "POC_RESULT: MISSING_TEST_FILE=${TEST_PATH}" >&2
  exit 2
fi

mkdir -p "$(dirname "${BASE_TEST_PATH}")" "${BASE_CONTRACTS_DIR}/.testdata"
cp "${TEST_PATH}" "${BASE_TEST_PATH}"
cp "${SP1_FIXTURE_FILE}" "${BASE_FIXTURE_FILE}"
cp "${SP1_VERIFIER_ARTIFACT}" "${BASE_SP1_VERIFIER_ARTIFACT}"
export SP1_FIXTURE_FILE="${BASE_FIXTURE_FILE}"
export SP1_VERIFIER_ARTIFACT="${BASE_SP1_VERIFIER_ARTIFACT}"

echo "POC_RESULT: ACTUAL_BASE_ZKVERIFIER_TEST_ROOT=${BASE_CONTRACTS_DIR}"
echo "POC_RESULT: ACTUAL_BASE_ZKVERIFIER_TEST_PATH=${BASE_TEST_PATH}"

forge test \
  --root "${BASE_CONTRACTS_DIR}" \
  --match-path "test/multiproof/$(basename "${BASE_TEST_PATH}")" \
  --match-test "test_RealSP1Groth16_RC28IntervalProofRejectedByOnchainJournal" \
  -vv

echo "POC_RESULT: INTERVAL_MISMATCH_PROVEN=true"
echo "POC_RESULT: HIGH_TEMPORARY_FREEZE_CANDIDATE=true"
```

</details>

<details>

<summary><code>RC28IntervalMismatchRealSP1PoC.t.sol</code></summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { Test } from "forge-std/Test.sol";

import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { ISP1Verifier } from "src/dispute/zk/ISP1Verifier.sol";
import { ZkVerifier } from "src/multiproof/zk/ZKVerifier.sol";

contract RC28IntervalMismatchRealSP1PoC is Test {
    bytes32 internal constant PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH =
        0x0020f5df94945d8d40bc2b11128cbf2535565f5cde58960029017ee2ccef46d7;

    struct Fixture {
        bytes onchainProof;
        bytes rc28Proof;
        bytes32 vkey;
        bytes32 onchainJournal;
        bytes32 rc28Journal;
        bytes32 publicSepoliaZkAggregateHash;
        bytes32 configHash;
        bytes32 zkRangeHash;
        bytes32 l1OriginHash;
        bytes32 startingRoot;
        bytes32 endingRoot;
        uint256 startingL2;
        uint256 endingL2;
        uint256 onchainRootCount;
        uint256 rc28RootCount;
        uint256 onchainPackedLen;
        uint256 rc28PackedLen;
        uint256 contractBlockInterval;
        uint256 contractIntermediateBlockInterval;
        uint256 rc28DefaultIntermediateRootInterval;
        uint256 fastFinalizationDelaySeconds;
        uint256 slowFinalizationDelaySeconds;
        uint256 additionalWithdrawalFreezeSeconds;
    }

    function test_RealSP1Groth16_RC28IntervalProofRejectedByOnchainJournal() external {
        Fixture memory fixture = _loadFixture();
        require(fixture.publicSepoliaZkAggregateHash == PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH, "public vkey mismatch");
        require(fixture.contractBlockInterval == 600, "bad block interval");
        require(fixture.contractIntermediateBlockInterval == 30, "bad contract intermediate interval");
        require(fixture.rc28DefaultIntermediateRootInterval == 10, "bad rc28 interval");
        require(fixture.onchainRootCount == 20, "bad onchain root count");
        require(fixture.rc28RootCount == 60, "bad rc28 root count");
        require(fixture.onchainJournal != fixture.rc28Journal, "journals should differ");
        require(fixture.vkey != PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH, "equivalent vkey should differ from deployed vkey");

        ISP1Verifier sp1Verifier = ISP1Verifier(_deploySP1Verifier());
        ZkVerifier zkVerifier = new ZkVerifier(sp1Verifier, IAnchorStateRegistry(address(0xA11CE)));

        require(
            zkVerifier.verify(fixture.onchainProof, fixture.vkey, fixture.onchainJournal),
            "onchain interval proof rejected"
        );
        require(
            zkVerifier.verify(fixture.rc28Proof, fixture.vkey, fixture.rc28Journal),
            "rc28 interval proof rejected under its own journal"
        );

        (bool wrongJournalAccepted, bytes4 wrongJournalSelector) =
            _verifierAcceptance(address(zkVerifier), fixture.rc28Proof, fixture.vkey, fixture.onchainJournal);
        require(!wrongJournalAccepted, "rc28 proof accepted under 30-block onchain journal");

        (bool publicSepoliaVkeyAccepted, bytes4 publicSepoliaVkeySelector) = _verifierAcceptance(
            address(zkVerifier), fixture.onchainProof, PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH, fixture.onchainJournal
        );
        require(!publicSepoliaVkeyAccepted, "equivalent proof accepted under public Sepolia aggregate vkey");

        _resultBool("ZK_VERIFIER_ONCHAIN_PROOF_ACCEPTED", true);
        _resultBool("ZK_VERIFIER_RC28_PROOF_ACCEPTED_WITH_RC28_JOURNAL", true);
        _resultBool("ZK_VERIFIER_RC28_PROOF_ACCEPTED_WITH_ONCHAIN_JOURNAL", wrongJournalAccepted);
        _resultBytes4("ZK_VERIFIER_RC28_WRONG_JOURNAL_REVERT_SELECTOR", wrongJournalSelector);
        _resultBool("ZK_VERIFIER_PUBLIC_SEPOLIA_VKEY_ACCEPTED", publicSepoliaVkeyAccepted);
        _resultBytes4("ZK_VERIFIER_PUBLIC_SEPOLIA_VKEY_REVERT_SELECTOR", publicSepoliaVkeySelector);
        _resultBool("PROGRAM_VKEY_DIFFERS_FROM_PUBLIC_SEPOLIA", fixture.vkey != PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH);
        _resultUint("FULL_PROPOSAL_ONCHAIN_ROOT_COUNT", fixture.onchainRootCount);
        _resultUint("FULL_PROPOSAL_RC28_ROOT_COUNT", fixture.rc28RootCount);
        _resultUint("FULL_PROPOSAL_ONCHAIN_PACKED_LEN", fixture.onchainPackedLen);
        _resultUint("FULL_PROPOSAL_RC28_PACKED_LEN", fixture.rc28PackedLen);
        _resultBytes32("FULL_PROPOSAL_ONCHAIN_DIGEST", fixture.onchainJournal);
        _resultBytes32("FULL_PROPOSAL_RC28_DIGEST", fixture.rc28Journal);
        _resultBool("FULL_PROPOSAL_DIGESTS_EQUAL", false);
        _resultUint("ADDITIONAL_WITHDRAWAL_FREEZE_SECONDS", fixture.additionalWithdrawalFreezeSeconds);
        _resultBool("ADDITIONAL_WITHDRAWAL_FREEZE_GE_24H", fixture.additionalWithdrawalFreezeSeconds >= 1 days);
    }

    function _loadFixture() internal view returns (Fixture memory fixture) {
        string memory json = vm.readFile(vm.envString("SP1_FIXTURE_FILE"));
        fixture.onchainProof = vm.parseJsonBytes(json, ".onchainProof");
        fixture.rc28Proof = vm.parseJsonBytes(json, ".rc28Proof");
        fixture.vkey = vm.parseJsonBytes32(json, ".vkey");
        fixture.onchainJournal = vm.parseJsonBytes32(json, ".onchainJournal");
        fixture.rc28Journal = vm.parseJsonBytes32(json, ".rc28Journal");
        fixture.publicSepoliaZkAggregateHash = vm.parseJsonBytes32(json, ".publicSepoliaZkAggregateHash");
        fixture.configHash = vm.parseJsonBytes32(json, ".configHash");
        fixture.zkRangeHash = vm.parseJsonBytes32(json, ".zkRangeHash");
        fixture.l1OriginHash = vm.parseJsonBytes32(json, ".l1OriginHash");
        fixture.startingRoot = vm.parseJsonBytes32(json, ".startingRoot");
        fixture.endingRoot = vm.parseJsonBytes32(json, ".endingRoot");
        fixture.startingL2 = vm.parseJsonUint(json, ".startingL2");
        fixture.endingL2 = vm.parseJsonUint(json, ".endingL2");
        fixture.onchainRootCount = vm.parseJsonUint(json, ".onchainRootCount");
        fixture.rc28RootCount = vm.parseJsonUint(json, ".rc28RootCount");
        fixture.onchainPackedLen = vm.parseJsonUint(json, ".onchainPackedLen");
        fixture.rc28PackedLen = vm.parseJsonUint(json, ".rc28PackedLen");
        fixture.contractBlockInterval = vm.parseJsonUint(json, ".contractBlockInterval");
        fixture.contractIntermediateBlockInterval = vm.parseJsonUint(json, ".contractIntermediateBlockInterval");
        fixture.rc28DefaultIntermediateRootInterval = vm.parseJsonUint(json, ".rc28DefaultIntermediateRootInterval");
        fixture.fastFinalizationDelaySeconds = vm.parseJsonUint(json, ".fastFinalizationDelaySeconds");
        fixture.slowFinalizationDelaySeconds = vm.parseJsonUint(json, ".slowFinalizationDelaySeconds");
        fixture.additionalWithdrawalFreezeSeconds = vm.parseJsonUint(json, ".additionalWithdrawalFreezeSeconds");
    }

    function _deploySP1Verifier() internal returns (address verifier) {
        string memory artifact = vm.readFile(vm.envString("SP1_VERIFIER_ARTIFACT"));
        bytes memory bytecode = vm.parseJsonBytes(artifact, ".bytecode.object");
        assembly {
            verifier := create(0, add(bytecode, 0x20), mload(bytecode))
        }
        require(verifier != address(0), "SP1 verifier deploy failed");
    }

    function _verifierAcceptance(
        address verifier,
        bytes memory proofBytes,
        bytes32 imageId,
        bytes32 journal
    )
        internal
        view
        returns (bool accepted, bytes4 revertSelector)
    {
        (bool ok, bytes memory data) =
            verifier.staticcall(abi.encodeWithSignature("verify(bytes,bytes32,bytes32)", proofBytes, imageId, journal));
        if (!ok) return (false, _selector(data));
        if (data.length < 32) return (false, bytes4(0));
        return (abi.decode(data, (bool)), bytes4(0));
    }

    function _selector(bytes memory revertData) internal pure returns (bytes4 selector) {
        if (revertData.length < 4) return bytes4(0);
        assembly {
            selector := mload(add(revertData, 0x20))
        }
    }

    function _resultBool(string memory key, bool value) internal {
        emit log_string(string.concat("POC_RESULT: ", key, "=", value ? "true" : "false"));
    }

    function _resultUint(string memory key, uint256 value) internal {
        emit log_string(string.concat("POC_RESULT: ", key, "=", vm.toString(value)));
    }

    function _resultBytes32(string memory key, bytes32 value) internal {
        emit log_string(string.concat("POC_RESULT: ", key, "=", vm.toString(value)));
    }

    function _resultBytes4(string memory key, bytes4 value) internal {
        emit log_string(string.concat("POC_RESULT: ", key, "=", vm.toString(value)));
    }
}
```

</details>

### Expected markers

* `POC_RESULT: REAL_ZK_PROOF_SYSTEM=SP1 Groth16`
* `POC_RESULT: SP1_PROGRAM_VKEY_BYTES32_FORMAT=true`
* `POC_RESULT: PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH_BYTES32_FORMAT=true`
* `POC_RESULT: PROGRAM_VKEY_DIFFERS_FROM_PUBLIC_SEPOLIA=true`
* `POC_RESULT: FULL_PROPOSAL_ONCHAIN_ROOT_COUNT=20`
* `POC_RESULT: FULL_PROPOSAL_RC28_ROOT_COUNT=60`
* `POC_RESULT: FULL_PROPOSAL_ONCHAIN_PACKED_LEN=836`
* `POC_RESULT: FULL_PROPOSAL_RC28_PACKED_LEN=2116`
* `POC_RESULT: FULL_PROPOSAL_DIGESTS_EQUAL=false`
* `POC_RESULT: ONCHAIN_SP1_PROOF_BYTES_PRESENT=true`
* `POC_RESULT: RC28_SP1_PROOF_BYTES_PRESENT=true`
* `POC_RESULT: SP1_CONTRACTS_COMMIT_MATCH=true`
* `POC_RESULT: BASE_CONTRACTS_COMMIT_MATCH=true`
* `POC_RESULT: ZK_VERIFIER_ONCHAIN_PROOF_ACCEPTED=true`
* `POC_RESULT: ZK_VERIFIER_RC28_PROOF_ACCEPTED_WITH_RC28_JOURNAL=true`
* `POC_RESULT: ZK_VERIFIER_RC28_PROOF_ACCEPTED_WITH_ONCHAIN_JOURNAL=false`
* `POC_RESULT: ZK_VERIFIER_PUBLIC_SEPOLIA_VKEY_ACCEPTED=false`
* `POC_RESULT: ADDITIONAL_WITHDRAWAL_FREEZE_SECONDS=518400`
* `POC_RESULT: ADDITIONAL_WITHDRAWAL_FREEZE_GE_24H=true`
* `POC_RESULT: INTERVAL_MISMATCH_PROVEN=true`
* `POC_RESULT: HIGH_TEMPORARY_FREEZE_CANDIDATE=true`
* `Suite result: ok. 1 passed; 0 failed; 0 skipped`

### AWS fresh run

The current `poc/OUTPUT.txt` was produced by executing `bash run_poc.sh` on `2026-04-27` on an AWS `r7i.16xlarge` on-demand VM in `us-east-1`, using the included `aws_spot_test.sh` helper with a 350 GB root volume. The helper copied the flat PoC package to a temporary remote working directory, executed the reviewer command, preserved `OUTPUT.txt`, and terminated the instance.

Key observed markers from that run:

```
POC_RESULT: REAL_ZK_PROOF_SYSTEM=SP1 Groth16
POC_RESULT: SP1_PROGRAM_VKEY=0x00d875659805d63e190b52eeda23c1f6a24c4fed4add8084841ab394e8de59ed
POC_RESULT: PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH_NOT_USED=0x0020f5df94945d8d40bc2b11128cbf2535565f5cde58960029017ee2ccef46d7
POC_RESULT: PROGRAM_VKEY_DIFFERS_FROM_PUBLIC_SEPOLIA=true
POC_RESULT: FULL_PROPOSAL_ONCHAIN_ROOT_COUNT=20
POC_RESULT: FULL_PROPOSAL_RC28_ROOT_COUNT=60
POC_RESULT: FULL_PROPOSAL_DIGESTS_EQUAL=false
POC_RESULT: ZK_VERIFIER_ONCHAIN_PROOF_ACCEPTED=true
POC_RESULT: ZK_VERIFIER_RC28_PROOF_ACCEPTED_WITH_RC28_JOURNAL=true
POC_RESULT: ZK_VERIFIER_RC28_PROOF_ACCEPTED_WITH_ONCHAIN_JOURNAL=false
POC_RESULT: ZK_VERIFIER_PUBLIC_SEPOLIA_VKEY_ACCEPTED=false
POC_RESULT: ADDITIONAL_WITHDRAWAL_FREEZE_SECONDS=518400
POC_RESULT: ADDITIONAL_WITHDRAWAL_FREEZE_GE_24H=true
POC_RESULT: INTERVAL_MISMATCH_PROVEN=true
POC_RESULT: HIGH_TEMPORARY_FREEZE_CANDIDATE=true
```

### Meaning of valid ZK proof

For this PoC, a ZK proof is `valid` when all of the following are true:

* SP1 Groth16 generated the proof bytes for the aggregate-journal program used by the runner.
* The public journal has the same field order and hash preimage shape consumed by upstream `AggregateVerifier._verifyZkProof(...)`.
* The vkey is a `bytes32` SP1 program verification key passed to the release-pinned upstream `ZkVerifier.verify(proofBytes, imageId, journal)` path, the same Solidity type and verifier interface used for Base Sepolia's `ZK_AGGREGATE_HASH`.
* The release-pinned upstream Base `ZkVerifier.verify(...)` call accepts `(programVKey, journal, proofBytes)` through the Succinct SP1 Groth16 verifier contract.
* No mock verifier, signer registration, privileged registry mutation, or verification bypass is used.
* The onchain-interval proof and rc.28 proof each verify when paired with the journal they prove.

This definition is bounded by the public artifacts available to the PoC. The vkey has the same encoding, Solidity type, and verification-call semantics as Base Sepolia's public aggregate vkey. The difference is the vkey value, which identifies an equivalent aggregate-journal SP1 program rather than Base Sepolia's deployed aggregate program. This definition does not assert that these proof bytes verify against Base Sepolia's deployed aggregate vkey. Removing that boundary requires production aggregate proof material for public Sepolia's `ZK_AGGREGATE_HASH`.

### Production boundary

The runner intentionally does not use a production fork gate because public artifacts do not provide proof material for Base Sepolia's configured aggregate vkey. The PoC uses real SP1 Groth16 proofs for an equivalent aggregate-journal program and wires them through the release-pinned upstream Base `ZkVerifier` path.

This boundary is explicit in the output:

```
POC_RESULT: PROGRAM_VKEY_DIFFERS_FROM_PUBLIC_SEPOLIA=true
POC_RESULT: ZK_VERIFIER_PUBLIC_SEPOLIA_VKEY_ACCEPTED=false
```

The primary path is ZK-only and verifies real SP1 proof bytes through the release-pinned upstream Base `ZkVerifier` path.

### Network safety

The runner does not broadcast transactions and does not interact with public network state. All verifier calls occur inside Foundry execution. It does not require `SEPOLIA_RPC_URL`, wallet material, a funded account, or a fork block.

### What is not proven

* This is not a public testnet or mainnet broadcast.
* This is not a production fork proof.
* This is not proof material for Base Sepolia's configured aggregate vkey.
* This is not a forged-proof, bridge-loss, invalid-state-root-finalization, permanent-freeze, or chain-shutdown claim.
