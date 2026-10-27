# 76386 bc high time controlled disputegamefactory create pins game l1head to a batch incomplete l1 block making early halt finalization on chain unchallengeable

**Submitted on May 4th 2026 at 06:01:33 UTC by @QED for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76386
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization

## Description

## Brief/Intro

The SP1 range program does not bind the public-values `l2BlockNumber` to the actual derivation endpoint reached by the guest. When derivation hits `EndOfSource`, `advance_to_target` retargets the requested block to the current safe head and continues; the executor epilogue checks only `output_root == boot.claimed_l2_output_root`, never `safe_head.block_info.number == boot.claimed_l2_block_number`; `BootInfoStruct::new` then commits the inflated `claimed_l2_block_number = N` to the journal regardless of what block execution actually reached.

The on-chain `AggregateVerifier` correctly pins the journal-bound `l1OriginHash` to the immutable `game.l1Head()` set at `DisputeGameFactory.create()` time (`blockhash(block.number - 1)` with no batch-coverage check). A malicious proposer therefore times `create()` to land in an L1 block whose history does **not** yet contain the L2 batches required to derive up to the contract-pinned `endingL2SequenceNumber = N`; the attacker's SP1 proof exits cleanly with `output_root = root@M` for some `M < N`, the contract's calldata invariants pass, and the game initializes. Critically, **any honest counter-proof must commit a journal binding the same fixed `l1OriginHash`**, and under that l1Head no derivation reaches block `N` — the honest challenger's executor epilogue rejects every candidate `claimed_l2_output_root = root@N` because their guest only reaches `root@M`. The dispute/challenge mechanism is structurally impossible to invoke; the malicious proposal resolves DEFENDER\_WINS by construction.

## Vulnerability Details

**Severity: Critical** -- "Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization"

The defect is the composition of (a) a program-logic gap in the SP1 range guest that lets the prover commit a journal claiming end-block `N` while derivation halted at `M < N`, and (b) the on-chain `AggregateVerifier` design that binds the journal's `l1OriginHash` to the proposer-controlled `game.l1Head()`. Either layer alone would be a defense-in-depth gap; together they remove the only on-chain remediation path.

### Root Cause

{% stepper %}
{% step %}

## The `EndOfSource` arm in `advance_to_target` retargets to the safe head and continues, instead of failing.

[`crates/succinct/utils/client/src/client.rs:93-115`](https://github.com/base/base/blob/48d5370229de0403b7b5defe20e476b4690e3bf4/crates/succinct/utils/client/src/client.rs#L93-L115) (verbatim, v0.8.0-rc.35):

```rust
let mut attributes = match driver.pipeline.produce_payload(tip_cursor.l2_safe_head).await {
    Ok(attrs) => attrs.take_inner(),
    Err(PipelineErrorKind::Critical(PipelineError::EndOfSource)) => {
        warn!(target: "client", "Exhausted data source; Halting derivation and using current safe head.");

        // Adjust the target block number to the current safe head, as no more blocks
        // can be produced.
        if target.is_some() {
            target = Some(tip_cursor.l2_safe_head.block_info.number);  // <-- silently rewrites the prover-supplied target to whatever block was reached
        };

        // If we are in interop mode, this error must be handled by the caller.
        // Otherwise, we continue the loop to halt derivation on the next iteration.
        if cfg.is_isthmus_active(driver.cursor.read().l2_safe_head().block_info.number) {  // <-- guard receives a block NUMBER where the function expects a UNIX TIMESTAMP
            return Err(PipelineError::EndOfSource.crit().into());
        }
        continue;
    }
    Err(e) => {
        error!(target: "client", "Failed to produce payload: {:?}", e);
        return Err(DriverError::Pipeline(e));
    }
};
```

{% endstep %}

{% step %}

## The Isthmus guard intended to block this path is permanently inactive on Base mainnet because of a type confusion.

[`crates/consensus/genesis/src/rollup.rs:217-220`](https://github.com/base/base/blob/48d5370229de0403b7b5defe20e476b4690e3bf4/crates/consensus/genesis/src/rollup.rs#L217-L220):

```rust
pub fn is_isthmus_active(&self, timestamp: u64) -> bool {  // <-- parameter is documented and used as a Unix timestamp
    self.hardforks.isthmus_time.is_some_and(|t| timestamp >= t)
        || self.is_jovian_active(timestamp)
}
```

Base mainnet's `isthmus_timestamp = 1_746_806_401` (Unix). Block numbers on Base mainnet are currently \~46M. The guard at `client.rs:106` evaluates `block_number >= timestamp`, which is `46_000_000 >= 1_746_806_401 = false` for every realistic Base mainnet block height. The early-halt error is never re-raised; the loop continues with `target` shrunk to whatever block was actually reached.
{% endstep %}

{% step %}

## The executor epilogue checks `output_root` only, not the safe-head block number.

[`crates/succinct/utils/client/src/witness/executor.rs:158-180`](https://github.com/base/base/blob/48d5370229de0403b7b5defe20e476b4690e3bf4/crates/succinct/utils/client/src/witness/executor.rs#L158-L180):

```rust
let (safe_head, output_root, intermediate_roots) = advance_to_target(
    &mut driver,
    rollup_config.as_ref(),
    Some(boot.claimed_l2_block_number),
    intermediate_root_interval,
)
.await?;

////////////////////////////////////////////////////////////////
//                          EPILOGUE                          //
////////////////////////////////////////////////////////////////

if output_root != boot.claimed_l2_output_root {  // <-- only output-root equality is enforced
    return Err(anyhow!(
        "Failed to validate L2 block #{number} with claimed output root {claimed_output_root}. Got {output_root} instead",
        number = safe_head.block_info.number,
        output_root = output_root,
        claimed_output_root = boot.claimed_l2_output_root,
    ));
}
// <-- NO assertion that safe_head.block_info.number == boot.claimed_l2_block_number
```

{% endstep %}

{% step %}

## `BootInfoStruct::new` propagates the inflated `claimed_l2_block_number` into the public-values journal, regardless of what block execution actually reached.

[`crates/succinct/utils/client/src/boot.rs:37-58`](https://github.com/base/base/blob/48d5370229de0403b7b5defe20e476b4690e3bf4/crates/succinct/utils/client/src/boot.rs#L37-L58):

```rust
pub fn new(
    boot_info: BootInfo,
    l2_pre_block_number: u64,
    intermediate_roots: Vec<B256>,
) -> Self {
    Self {
        l1Head: boot_info.l1_head,
        l2PreRoot: boot_info.agreed_l2_output_root,
        l2PostRoot: boot_info.claimed_l2_output_root,
        l2PreBlockNumber: l2_pre_block_number,
        l2BlockNumber: boot_info.claimed_l2_block_number,  // <-- copied verbatim from prover input; never compared to the block actually reached
        rollupConfigHash: hash_rollup_config(&boot_info.rollup_config),
        intermediateRoots: Bytes::from(
            intermediate_roots
                .iter()
                .flat_map(|root| root.as_slice())
                .copied()
                .collect::<Vec<u8>>(),
        ),
    }
}
```

{% endstep %}

{% step %}

## The on-chain `AggregateVerifier` pins the journal-bound `l1OriginHash` to the immutable `game.l1Head()`.

[`src/multiproof/AggregateVerifier.sol:904-944`](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L904-L944) (v8.1.0):

```solidity
function _verifyZkProof(
    bytes calldata proofBytes,
    address proposer,
    bytes32 l1OriginHash,         // <-- caller of _verifyZkProof always passes l1Head().raw()
    bytes32 startingRoot,
    uint64 startingL2SequenceNumber,
    bytes32 endingRoot,
    uint64 endingL2SequenceNumber,
    bytes memory intermediateRoots
)
    internal
    view
{
    bytes32 journal = keccak256(
        abi.encodePacked(
            proposer,
            l1OriginHash,              // <-- bound to game's stored l1Head; SP1 verifier must match this exact value
            startingRoot,
            startingL2SequenceNumber,
            endingRoot,
            endingL2SequenceNumber,
            intermediateRoots,
            CONFIG_HASH,
            ZK_RANGE_HASH
        )
    );

    // Validate the proof.
    if (!ZK_VERIFIER.verify(proofBytes, ZK_AGGREGATE_HASH, journal)) revert InvalidProof();
}
```

[`src/multiproof/AggregateVerifier.sol:548-588`](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L548-L588) (`nullify` — same shape as `challenge`):

```solidity
function nullify(
    bytes calldata proofBytes,
    uint256 intermediateRootIndex,
    bytes32 intermediateRootToProve
)
    external
{
    ...
    (bytes32 startingRoot, uint64 startingL2SequenceNumber, uint64 endingL2SequenceNumber) =
        _getStartingIntermediateRootAndL2SequenceNumbers(intermediateRootIndex);

    _verifyProof(
        proofBytes[1:],
        proofType,
        msg.sender,
        l1Head().raw(),           // <-- l1Head is fixed at game creation; cannot be widened by any later caller
        startingRoot,
        startingL2SequenceNumber,
        intermediateRootToProve,
        endingL2SequenceNumber,
        abi.encodePacked(intermediateRootToProve)
    );
    ...
}
```

[`src/multiproof/AggregateVerifier.sol:963-983`](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L963-L983):

```solidity
function _verifyL1Origin(bytes32 l1OriginHash, uint256 l1OriginNumber) internal view {
    if (l1OriginNumber >= block.number) {
        revert L1OriginInFuture(l1OriginNumber, block.number);
    }

    bytes32 actualHash;
    uint256 blockAge = block.number - l1OriginNumber;

    // Prefer blockhash() over EIP-2935 when possible since it's cheaper (no external call).
    if (blockAge <= BLOCKHASH_WINDOW) {
        actualHash = blockhash(l1OriginNumber);    // <-- only existence-of-L1-block check; NO check that this L1 block contains enough batches to derive endingL2SequenceNumber
    } else {
        actualHash = HISTORY_STORAGE_CONTRACT.staticcall(
            abi.encodeWithSignature("get(uint256)", l1OriginNumber)
        );
    }
    if (actualHash != l1OriginHash) revert InvalidL1OriginHash(l1OriginHash, actualHash);
}
```

{% endstep %}

{% step %}

## The proposer chooses `game.l1Head()` by timing `DisputeGameFactory.create()`.

The factory pins `parentHash = blockhash(block.number - 1)` at `create()` time and never validates that this L1 block contains enough batch-publishing transactions to derive up to the protocol-pinned `endingL2SequenceNumber`. (Cited as background — `DisputeGameFactory` is OP-stack-standard and stores `parentHash` into the cloned `IDisputeGame`'s immutable args at `[52, 84)`, where the multiproof game reads it back via `l1Head() = _getArgBytes32(0x34)` at [`AggregateVerifier.sol:748`](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L748).)
{% endstep %}
{% endstepper %}

### Why the asymmetry makes detection structurally impossible

Given a game whose stored `l1Head()` points to L1 block `H` such that the canonical L1 history at `H` contains batches sufficient to derive L2 to `M` but not to `N` (with `M < N` and `endingL2SequenceNumber = N` pinned by `_getStartingIntermediateRootAndL2SequenceNumbers`):

* **Attacker's proof:** runs the guest with witness preimages tied to `H`; derivation hits `EndOfSource` at `M`; `client.rs:101` rewrites `target = M`; the loop returns `(safe_head@M, output_root@M)`; `executor.rs:172` accepts because the prover supplied `claimed_l2_output_root = root@M`; `BootInfoStruct::new` commits journal `{l2BlockNumber: N, l2PostRoot: root@M, l1Head: H}`; SP1 verifies; `nullify`/`challenge` accepts.
* **Honest challenger's would-be counter-proof:** must commit to a journal with `endingRoot = root@N` (the canonical root, in order to win the challenge). Their guest *also* runs against witness data tied to the contract-pinned `l1Head = H`. Their guest *also* hits `EndOfSource` at `M` and produces `output_root = root@M`. The epilogue check at `executor.rs:172` then evaluates `output_root (root@M) != claimed_l2_output_root (root@N)` and the executor returns the validation error. **No valid SP1 proof committing `endingRoot = root@N` for journal-bound `l1OriginHash = H` exists**, because no execution under that l1Head reaches state `N`.

The off-chain detection is unaffected — an honest party reading canonical L2 via a node sees `intermediateOutputRoot(K-1) = root@M ≠ canonical root@N` and knows the proposal is invalid. The on-chain remediation, however, requires producing an SP1 proof binding `l1OriginHash = H ∧ endingRoot = root@N`, which is unsatisfiable. The `nullify` and `challenge` paths cannot be successfully invoked.

After the `SLOW_FINALIZATION_DELAY` window elapses with no successful nullification, the game resolves DEFENDER\_WINS. The pair `(l2SequenceNumber = N, rootClaim = root@M)` becomes the canonical anchor; subsequent games inherit the corrupted state.

### Attacker Model

* **Position:** Unauthenticated. Any L1 EOA or contract with sufficient ETH to post `initBonds[<gameType>]` can call `DisputeGameFactory.create()` (`external payable`, no auth modifier) and `MultiProofDisputeGame.initializeWithInitData()` (`external payable virtual`, no auth modifier). No proposer whitelist, allowlist, or staking gate; no signature, attestation, or registry membership required.
* **Trigger:** Submit `create(...)` such that it lands in L1 block `T` where `blockhash(T-1)` is an L1 block whose history (i.e., `0..T-1`) does not yet contain the batch-publishing transactions for L2 blocks in `(M, N]`, where `N = startingL2SequenceNumber + BLOCK_INTERVAL` is fixed by the previous anchor.
* **Economic cost:** `initBonds[<gameType>]` for the proposer slot. This bond is *recovered* on DEFENDER\_WINS, which is the deterministic outcome of this attack — net cost is gas + SP1 proving cost.
* **Precondition(s):**
  * L2 sequencer-to-L1 batch-publishing lag spans a non-zero window (universally true on production OP-stack chains; observable on Base mainnet — see live probe below).
  * Attacker is the first proposer to call `create()` for the next sealed window (one race per `BLOCK_INTERVAL` window; honest proposers normally wait for batches to land before creating).
  * Multiproof game type registered as `respectedGameType()` on `OptimismPortal`.

### Sequence

```mermaid
sequenceDiagram
    autonumber
    participant Att as Attacker (proposer)
    participant L2 as L2 sequencer
    participant L1 as Ethereum L1
    participant DGF as DisputeGameFactory
    participant Game as MultiProof Game
    participant Chal as Honest Challenger

    L2->>L1: publish batches for L2 blocks ..M  (lands in L1 block T-K)
    Note over L2,L1: batches for L2 (M, N] pending; not yet in L1
    Att->>L1: tx(DGF.create(rootClaim=root@M, extraData={l2BlockNumber=N})) high gas
    L1->>DGF: include in block T
    DGF->>Game: clone with l1Head = blockhash(T-1)  -- batch-incomplete history
    Note over Game: l1Head() permanently returns blockhash(T-1)
    L2->>L1: batches for (M, N] land in block T+1, T+2, ...  (irrelevant; game.l1Head is frozen)

    Att->>Att: build SP1 proof under witness keyed to l1Head=blockhash(T-1)
    Note over Att: guest runs, hits EndOfSource at M, retargets to M, exits Ok with output_root=root@M
    Att->>Game: submit proof; game initialized; bond posted

    Note over Chal: 7-day window opens
    Chal->>Chal: read canonical L2 via local node; compute root@N; observe proposed root@M ≠ root@N
    Chal->>Chal: try to build SP1 counter-proof for journal {endingRoot=root@N, l1OriginHash=blockhash(T-1)}
    Note over Chal: guest hits EndOfSource at M; output_root=root@M ≠ claimed=root@N;<br/>executor.rs:172 returns Err; NO valid proof exists
    Chal--xGame: cannot call nullify(proof, K-1, root@N) — no proof to submit

    Note over Game: SLOW_FINALIZATION_DELAY elapses with no nullification
    Game->>Game: resolve() -> DEFENDER_WINS
    Game-->>Att: bond returned; (l2SequenceNumber=N, rootClaim=root@M) finalized as anchor
```

## Impact Details

**Live trigger probe (2026-05-04):**

```bash
$ curl -s https://mainnet.base.org -X POST -H 'content-type: application/json' \
    -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' | jq -r .result
0x2b6d671   # block 45,602,417 — far below isthmus_time = 1,746,806,401
```

`is_isthmus_active(45_602_417)` evaluates `45_602_417 >= 1_746_806_401` -> `false`. The Isthmus guard at `client.rs:106` is therefore non-functional for every current Base mainnet block height; the EndOfSource fallback path at `client.rs:95-115` is reachable on every range proof at any Base mainnet block.

**Live trigger probe — sequencer-batch lag window (2026-05-04):**

L1 batch-publishing transactions to Base's batch-inbox lag the L2 head by O(seconds) on production. Each `create(...)` call lands in some L1 block T; for T to satisfy the attack precondition, `blockhash(T-1)` must precede the L1 inclusion of batches for the L2 range `(M, N]`. Because batches are published in discrete L1 transactions interleaved with arbitrary other L1 traffic, there is at minimum one "gap" L1 block per batch-publishing cycle in which `blockhash(T-1)` does not yet contain the latest L2 batches. The attacker need only race honest proposers into one such gap.

### What the bug enables

A proposer with the multiproof initialization bond can finalize a forged `(l2SequenceNumber=N, rootClaim=root@M)` anchor by exploiting the asymmetry: their proof passes the executor epilogue (`output_root@M == claimed_l2_output_root=root@M`) while any honest counter-proof attempting to bind `claimed_l2_output_root=root@N` is rejected by the same epilogue (`output_root@M != claimed_l2_output_root=root@N`).

The downstream consumer of a finalized anchor is the L1 withdrawal-finalization path through `OptimismPortal`, which validates withdrawal proofs against the finalized output root. A corrupted anchor of the form `(l2SequenceNumber=N, rootClaim=root@M)` causes:

[`crates/succinct/utils/client/src/witness/executor.rs:172-180`](https://github.com/base/base/blob/48d5370229de0403b7b5defe20e476b4690e3bf4/crates/succinct/utils/client/src/witness/executor.rs#L172-L180) (downstream consumer of the inflated `claimed_l2_block_number`):

```rust
if output_root != boot.claimed_l2_output_root {  // <-- under fixed game.l1Head, this check is the wall that prevents honest counter-proofs from being generated
    return Err(anyhow!(
        "Failed to validate L2 block #{number} with claimed output root {claimed_output_root}. Got {output_root} instead",
        number = safe_head.block_info.number,            // <-- safe_head is M
        output_root = output_root,                       // <-- output_root is root@M
        claimed_output_root = boot.claimed_l2_output_root,  // <-- challenger's required claim is root@N -> Err
    ));
}
```

[`src/multiproof/AggregateVerifier.sol:560-572`](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L560-L572) (where the corrupted anchor is consumed by the post-challenge resolution path; quoted from `nullify`):

```solidity
if (counteredByIntermediateRootIndexPlusOne > 0) {
    if (intermediateRootIndex != counteredByIntermediateRootIndexPlusOne - 1) {
        revert InvalidIntermediateRootIndex();
    }
    if (intermediateRootToProve != intermediateOutputRoot(intermediateRootIndex)) {
        revert IntermediateRootMismatch(intermediateRootToProve, intermediateOutputRoot(intermediateRootIndex));   // <-- honest challenger never reaches this point because they cannot produce a valid SP1 proof for root@N
    }
    if (proofType != ProofType.ZK) revert InvalidProofType();
}
```

The corrupted anchor exposes any subsequent L1 withdrawal-finalization that validates against `outputRoot(N) = root@M` to acceptance of withdrawal proofs whose Merkle paths trace into block `M`'s `withdrawalStorageRoot` rather than block `N`'s. The chain split between L1's view of L2 state at sequence `N` (= `root@M`) and the canonical L2 state at block `N` cannot be reconciled by any subsequent on-chain action, since each future game must be built on top of the finalized (incorrect) anchor — and the same primitive used to corrupt anchor `K` can be re-applied to corrupt every subsequent anchor.

## Recommended Fix

1. **`crates/succinct/utils/client/src/witness/executor.rs` epilogue** — assert `safe_head.block_info.number == boot.claimed_l2_block_number` immediately after the `output_root` check, before returning `Ok((boot_clone, intermediate_roots))`. This is the load-bearing fix: it closes the bug regardless of whether (2) and (3) below are also applied, by ensuring the guest can never produce a journal whose `l2BlockNumber` is past where execution actually reached.
2. **`crates/succinct/utils/client/src/client.rs:106`** — pass `driver.cursor.read().l2_safe_head().block_info.timestamp` (a Unix timestamp) to `is_isthmus_active`, not `.number` (a block height). Restores the intended Isthmus guard. Alternatively, remove the `EndOfSource` fallback entirely so the loop propagates the error unconditionally.
3. **`crates/succinct/utils/client/src/boot.rs::BootInfoStruct::new`** — accept the actually-reached `safe_head.block_info.number` rather than `boot_info.claimed_l2_block_number`, or assert their equality at construction. Makes the inflation impossible to introduce at the journal-build site.

## Link to Proof of Concept

<https://gist.github.com/x-qedaudit/71fe7ea7918ee0c9867b3aa58a437520>

## Proof of Concept

**Runnable artifacts (secret gist):** <https://gist.github.com/x-qedaudit/71fe7ea7918ee0c9867b3aa58a437520>

### Setup

* Workspace-member crate at `etc/tools/poc_l1head_short_unchallengeable/` with `lib.rs` symlinked from `~/known-bugs/base/chain-validity-l1head-short-unchallengeable/poc_test.rs`.
* `ShortPipeline` is a real `base_consensus_derive::Pipeline + SignalReceiver + DriverPipeline` impl whose `step()` always returns `StepResult::StepFailed(PipelineErrorKind::Critical(PipelineError::EndOfSource))`. This is the attacker as a real type implementing the production pipeline trait — the same `MissingPreimageProvider` / `MaliciousOracle` idiom used elsewhere in this scope. The witness's L1 history reaches L2 block `M` only; the contract-pinned `endingL2SequenceNumber = N > M`.
* `StubExecutor` is a real `base_proof_driver::Executor` impl. `execute_payload` and `compute_output_root` are never called for these tests because EndOfSource fires on the first `produce_payload` call, but the type must be a real `Executor` to satisfy `Driver::new`.
* `epilogue_real` is a verbatim copy of `crates/succinct/utils/client/src/witness/executor.rs:172-180`. The static `epilogue_source_matches` test asserts the in-tree code matches the copy and that the missing block-number reconciliation is still absent.
* Both tests build the same `Driver` with `ShortPipeline`+`StubExecutor`+`PipelineCursor` seeded at `safe_head=M`, `safe_head_output_root=root@M`. They then call the real `Driver::advance_to_target(cfg, Some(N), |_,_|{})`. The two tests differ only in `boot.claimed_l2_output_root`.

### Observed asymmetry + on-chain journal-hash equivalence

* **Test A (`attacker_proof_passes_under_short_l1head`):** real `Driver::advance_to_target(target=N)` returns `Ok((safe_head=M, output_root=root@M))` via the EndOfSource silent-target-rewrite branch. With `boot.claimed_l2_output_root = root@M`, real `epilogue_real` returns `Ok`; real `BootInfoStruct::new` commits journal `{l2BlockNumber=N, l2PostRoot=root@M}`. **The forged proof is structurally generable.**
* **Test B (`honest_counter_proof_blocked_under_short_l1head`):** identical driver setup; real `Driver::advance_to_target(target=N)` again returns `Ok((safe_head=M, output_root=root@M))`. With `boot.claimed_l2_output_root = root@N` (the value an honest challenger would need to commit to invoke `nullify(zk_proof, K, root@N)`), real `epilogue_real` returns `Err("Failed to validate L2 block #M with claimed output root 0xbb...bb. Got 0xaa...aa instead")`. **No SP1 proof binding `{l2BlockNumber=N, l2PostRoot=root@N, l1OriginHash=game.l1Head()}` can be produced; nullify/challenge cannot be successfully invoked.**
* **Test C (`contract_journal_hash_matches_sp1_commitment`):** independently constructs (1) the SP1 aggregation program's committed digest = `keccak256(AggregationOutputs.abi_encode_packed())` for an attacker-crafted `AggregationOutputs{l2PostRoot=root@M, endingL2SequenceNumber=N, ...}` (per `crates/proof/succinct/programs/aggregation/src/main.rs:117`), and (2) the contract-assembled `_verifyZkProof` journal hash = `keccak256(abi.encodePacked(proposer, l1OriginHash, ..., CONFIG_HASH, ZK_RANGE_HASH))` (per `src/multiproof/AggregateVerifier.sol:920-927` v8.1.0) for the same field values. Asserts the two digests are byte-equal. **Bridges Tests A/B to the on-chain consequence:** an attacker who succeeds at Test A produces a proof whose committed digest equals the journal hash the contract's `nullify(_, K, root@M)` would assemble; `SP1_VERIFIER.verifyProof` therefore accepts.

The four tests together exhibit the asymmetry through real production code: `Driver::advance_to_target` body, the `Pipeline`/`SignalReceiver`/`DriverPipeline` trait dispatch, the `EndOfSource → silent target rewrite` branch, the executor epilogue conditional, `BootInfoStruct::new`'s journal construction, and the SP1 aggregation program's `keccak256(AggregationOutputs.abi_encode_packed())` commitment format are all real; only the L1-data source (the witness behind the pipeline) is substituted, exactly as the attacker would substitute it on-chain. Test C closes the loop on-chain.

### Reproduction

```bash
cd ~/known-bugs/base/chain-validity-l1head-short-unchallengeable
./run_poc.sh
./check.sh
```

Expected verdict:

```
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

Test S (epilogue source matches simulated logic):              PASS
Test A (attacker proof passes under short l1Head):             PASS
Test B (honest counter-proof blocked under short l1Head):      PASS
Test C (contract journal hash matches SP1 commitment):         PASS
VERDICT: ASYMMETRY_DEMONSTRATED_AND_ON_CHAIN_BRIDGED
```
