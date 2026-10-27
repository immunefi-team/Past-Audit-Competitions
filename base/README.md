# Base

## Reports by Severity

<details>

<summary>Critical</summary>

* \#74480 \[BC-Critical] Gossip snappy decompression missing MAX\_GOSSIP\_SIZE check lets any peer DoS base-consensus
* \#74531 \[BC-Critical] Gossipsub message-id computation Snappy-decompresses untrusted payloads without a decompressed-size cap
* \#74391 \[BC-Critical] Missing Snappy Decoded-Length Bounds in CL Gossip Enable Batched Pre-Validation CPU / Allocator Churn
* \#75554 \[BC-Critical] Unbounded snappy decompression in NetworkPayloadEnvelope::decode\_v1..v4 leads to per-node memory exhaustion reachable from any unauthenticated P2P peer
* \#74657 \[BC-Critical] Remote Node DoS via Unbounded Snappy Decompression in Gossip Message Processing
* \#75101 \[BC-Critical] ZK Proof Executor Uses Wrong Azul Precompile Semantics, Allowing Proofs For Invalid L2 State Roots
* \#75305 \[BC-Critical] Unbounded Memory Allocation in Gossip Message Processing Allows Unauthenticated Attacker to Freeze Block Propagation
* \#75469 \[BC-Critical] Gossip payload decoder allocates unbounded Snappy output
* \#75485 \[BC-Critical] Unbounded Snappy Decompression in GossipSub \`message\_id\_fn\` Causes Per-Message Memory Spike of \~428 MiB Before Validation
* \#74656 \[BC-Critical] Gossip message\_id\_fn performs uncapped Snappy decompression before deduplication, enabling P2P memory-amplification DoS
* \#75301 \[BC-Critical] Attacker can DoS Consensus gossip via unbounded snappy decompression in message ID computation
* \#74513 \[BC-Critical] Unbounded snappy decompression in gossipsub message\_id\_fn causes pre-validation CPU and memory exhaustion from a mesh peer
* \#76096 \[BC-Critical] Unintended permanent chain split due to omitted \`min\_base\_fee\` validation in Jovian EIP-1559 implementation
* \#75962 \[BC-Critical] Pre-Validation Decompression Bomb in GossipSub Leads to Deterministic OOM (428 MiB/msg)
* \#75246 \[BC-Critical] Remote Snappy-compressed P2P block gossip message can force excessive memory and CPU usage in Base Azul nodes
* \#75653 \[BC-Critical] Missing operator fee in txpool validation allows costless pool pollution DoS
* \#74620 \[BC-Critical] Unbounded Snappy decompression in Base gossip message-id computation increases node resource consumption
* \#75504 \[BC-Critical] Snappy decompression bomb in gossipsub message handler crashes base nodes
* \#74556 \[BC-Critical] # Base Azul CL: unbounded snappy decompression in gossipsub \`message\_id\_fn\` allows a single unauthenticated P2P message to OOM-kill every reachable consensus node, halting the c...

</details>

<details>

<summary>High</summary>

* \#74944 \[BC-High] Flashblocks cached execution accepts skipped transactions — engine-level consensus split
* \#75938 \[BC-High] Span batch encoder omits genesis timestamp, causing all span batches to decode as future batches and be dropped post-Holocene
* \#76497 \[BC-High] Detached Task Semaphore Permit Leak in Consensus-Layer RPC Processor Starves EngineActor Router and Halts Block Production
* \#76500 \[BC-High] Batcher Reorg Reset Skips Canonical Blocks via Stale Safe Head
* \#76527 \[BC-High] Unauthenticated \`optimism\_outputAtBlock\` flood halts sequencer block production via EngineActor head-of-line blocking
* \#76386 \[BC-High] Time-controlled \`DisputeGameFactory.create()\` pins \`game.l1Head()\` to a batch-incomplete L1 block, making early-halt finalization on-chain unchallengeable
* \#76282 \[BC-High] EngineActor Single-Threaded Routing Blocked by RPC Flood Starves All Consensus Operations
* \#75710 \[BC-High] Cross-topic MessageId-cache poisoning blackholes block propagation in base-consensus-gossip
* \#75410 \[BC-High] Cross-topic message ID collision in gossipsub allows attacker to censor blocks from network nodes
* \#75683 \[BC-High] ProofsStorage silently drops newly written storage slots for destroyed-and-recreated accounts, breaking proof generation
* \#75826 \[BC-High] Off-by-one in batcher frame sizing causes infinite blob-encode hot loop, halting all L1 publication
* \#75533 \[BC-High] SP1 prover precompile map drops Jovian cap and Osaka pricing, enabling a forged \`outputRoot\` and ≥10% bridge drain wherever the ZK fast path is active
* \#76271 \[BC-High] Jovian minBaseFee is ignored during unsafe block consolidation, allowing divergent L2 headers
* \#75413 \[SC-High] ZK Proof Executor Derives BLOBBASEFEE From Base's DA-Footprint Header Field, Allowing Invalid Output Roots
* \#76229 \[BC-High] \`AttributesMatch::check\_eip1559\` Discards \`min\_base\_fee\` from Jovian Block Extra Data — Allows Network Processing Nodes to Process Mempool Transactions Beyond Canonical Set Parameters
* \#75371 \[BC-High] unlimited p2p connections lead to node isolation and RPC crash
* \#76494 \[BC-High] Batcher \`submit\_pending\` spin loop halts all L1 batch submission
* \#75868 \[BC-High] Unbounded Snappy Decompression in Public P2P Gossip Bypasses the 10 MiB Pre-Validation Limit
* \#75856 \[BC-High] Default blob batcher frame size can emit unencodable packed blob frames and delay L1 DA submission beyond 500%

</details>

<details>

<summary>Medium</summary>

* \#76155 \[SC-Medium] Proof Witness Holocene Fallback Can Relabel Witness Errors Into Deposit-Only State Roots
* \#76156 \[BC-Medium] Trailing DER alias bypasses Nitro certificate revocation
* \#76180 \[BC-Medium] Late frame confirmations can prune timed-out channels
* \#75950 \[BC-Medium] DestroyedChanged accounts make proof execution diverge from canonical L2 state or halt
* \#75935 \[BC-Medium] Unbounded Snappy Decompression in Gossip Message ID Computation Causes Pre-Validation Consensus Node Resource Exhaustion
* \#76136 \[BC-Medium] \`OnlineHostBackend::get\_preimage\` Contains an Unconditional Infinite Retry Loop
* \#74760 \[BC-Medium] State-root derivation divergence between base-reth-node and base-proof-executor lets unprivileged BLOBBASEFEE transactions stall TEE finality for affected L2 ranges
* \#74823 \[BC-Medium] min\_base\_fee - Consolidation bypass, below-floor tx inclusion + proof divergence
* \#76045 \[BC-Medium] Malformed batch will be retried as invalid payload attributes forever
* \#74791 \[BC-Medium] Stale \`intermediate\_block\_interval\` Cache Forces Dispute Games Into Incorrect Resolved State
* \#76065 \[SC-Medium] EndOfSource handling lets a ZK range proof claim an unreached L2 block
* \#76470 \[BC-Medium] Short ZK range proof can be accepted as a full AggregateVerifier interval and trigger global ZK verifier nullification
* \#75972 \[BC-Medium] ZK range proofs can stop at EndOfSource but still commit the requested target block, allowing a short-range proof to finalize an invalid longer-range output root
* \#74673 \[BC-Medium] Proof executor uses non-canonical BLOBBASEFEE after Jovian/Base V1
* \#75902 \[BC-Medium] Access Control Bypass in Consensus RPC 'admin\_postUnsafePayload' Can Lead to Chain Split via Skipped Isthmus Withdrawals-Root Validation
* \#76222 \[BC-Medium] Isthmus \`withdrawals\_root\` validation is skipped when the parent is only in the engine-tree overlay, allowing invalid multi-block unsafe/sync targets to be accepted
* \#74678 \[BC-Medium] Missing \`requests\_hash\` Validation in the Raw Import Path Can Persist Invalid Post-Isthmus Blocks and Cause Chain Split
* \#75623 \[BC-Medium] Post-Holocene span batch prefix validation can allow partially derived batches from a span rejected by full validation
* \#76232 \[BC-Medium] SP1 ZK proof validation accepts a claimed L2 block number that derivation never reached
* \#75911 \[BC-Medium] Base consensus gossip validates block payloads before signer verification, enabling remote CPU DoS
* \#75912 \[BC-Medium] Proof executor derives BLOBBASEFEE from Jovian DA footprint while Base nodes hard-code it to 1
* \#75354 \[BC-Medium] Hardcoded 10-block ZK witness interval makes 30-block challenge proofs unverifiable
* \#75709 \[BC-Medium] ZK range proofs can commit a later L2 block number while only deriving an earlier safe head, allowing stale output roots to be proven for the wrong L2 block
* \#75363 \[BC-Medium] ZK proof program uses a non-canonical precompile table, breaking soundness across every Base hardfork and enabling permanent chain split
* \#75369 \[BC-Medium] ZK range proof silently truncates derivation on \`EndOfSource\` and commits the original (T, R\_N) — permanent chain split when \`claimed\_l2\_block\_number\` outruns the sequenced L1 bat...
* \#75113 \[SC-Medium] ZK Challenge Proofs Use 10-Block Intermediate Roots While AggregateVerifier Verifies 30-Block Segments, Preventing ZK Correction of Invalid Proposals
* \#74569 \[BC-Medium] Base consensus client process-exits on safe \`INVALID\` payloads because \`is\_deposits\_only\` misclassifies every derived block and bypasses Holocene fallback
* \#76410 \[BC-Medium] Stateless \`GameScanner\` permanently drops invalid in-progress dispute games whose factory index falls below \`gameCount - lookback\_games\` after any single missed tick
* \#75390 \[BC-Medium] Invalid Derived Sequencer Txs Make EL Safe Roots Unprovable
* \#76107 \[BC-Medium] Gossip block validation performs the expensive payload clone, RLP re-encode and keccak before checking the cheap unsafe block signer signature, enabling sustained CPU and heap amp...
* \#76113 \[BC-Medium] Base ZK range proof can attest a 600-block checkpoint using only a partially derived EndOfSource range
* \#76425 \[BC-Medium] \`eth\_getProof\` response is not bound to \`L2ToL1MessagePasser\`, allowing a substituted bridge storage root to be accepted
* \#75979 \[BC-Medium] \`compute\_message\_id\` decompresses every incoming gossip message via \`snap::raw::Decoder::decompress\_vec\` without enforcing the documented decompressed-size cap, allowing per-messa...
* \#75986 \[BC-Medium] Proposer cold recovery loses the current anchor game address after ASR advances, causing subsequent multiproof games to revert
* \#75423 \[SC-Medium] ZK Range Client Accepts Truncated Post-Azul Execution, Allowing Invalid AggregateVerifier State Roots
* \#75992 \[BC-Medium] Cold-restart bootstrap break every honest proposer is permanently bricked after the first anchor advance, freezing L2→L1 withdrawals
* \#75719 \[BC-Medium] Proof executor unconditionally deletes destroyed-and-recreated accounts, producing invalid state roots that block L1 finalization
* \#75849 \[BC-Medium] Late challenge transitions can evade challenger scanning after aging out of the lookback window
* \#76454 \[BC-Medium] Range program \`EndOfSource\` guard dead, enabling trivial-proof finalization halt
* \#75437 \[BC-Medium] Jovian min\_base\_fee Omission Lets Consolidation Promote a Non-Canonical Unsafe Block as Safe
* \#75626 \[SC-Medium] rc28 OP-Succinct EndOfSource relabels an early output root as a later AggregateVerifier sequence
* \#75220 \[BC-Medium] Isthmus withdrawalsRoot validation skipped when parent state unavailable, allowing invalid blocks to pass post-execution validation
* \#74856 \[BC-Medium] Replay of stale sequencer-signed unsafe gossip rewinds unsafe\_head and causes unintended L2 reorg behavior
* \#75905 \[BC-Medium] Permissionless Bridge Withdrawal Halt via \`BLOBBASEFEE\` Divergence Between Reth and the Proof Program
* \#75249 \[SC-Medium] rc.28 ZK interval mismatch prevents dual-proof fast finality
* \#76536 \[BC-Medium] \[Medium]\[Blockchain/DLT] Proposer cold-start uses ASR sentinel as parent after anchor advancement, causing AggregateVerifier.initializeWithInitData to revert UnexpectedBlockNumber...
* \#75908 \[BC-Medium] ZK range proof does not bind the actual derived end block or intermediate-root cadence, allowing a shorter range to advance ASR as a full interval
* \#76075 \[BC-Medium] Proof executor unconditionally deletes destroyed-and-recreated accounts, producing invalid state roots that block L1 finalization
* \#74864 \[BC-Medium] Isthmus Withdrawals Root Validation Bypass Leading to Invalid Block Acceptance
* \#76294 \[BC-Medium] Isthmus withdrawals\_root validator silently accepts malformed blocks, producing peer divergence between honest Base nodes processing identical sequencer payloads (chain-level fork)
* \#75013 \[BC-Medium] DA backlog accounting misses encoded unpublished data, causing mempool processing beyond configured throttle limits
* \#75814 \[BC-Medium] Retryable span validation discards valid L1 batches and allows empty safe derivation
* \#75031 \[BC-Medium] Default blob frame sizing can create unencodable frames, causing retry storms and >30% batcher resource amplification
* \#75288 \[BC-Medium] ProofsStorage silently drops newly written storage slots for destroyed-and-recreated accounts, breaking proof generation
* \#76404 \[BC-Medium] Stateless challenger lookback window allows unbounded eviction of pending invalid proposals via DisputeGameFactory.create() spam
* \#75673 \[BC-Medium] Base V1 ZK proofs execute the old P256 precompile, allowing valid SP1 proofs for invalid output roots
* \#75686 \[SC-Medium] L1 Source Exhaustion Lets Range Proofs Finalize a Future L2 Block With an Old Output Root
* \#75527 \[BC-Medium] ProofsStorage silently drops newly written storage slots for destroyed-and-recreated accounts, breaking proof generation
* \#75757 \[BC-Medium] Unbounded snappy decompression in NetworkPayloadEnvelope::decode\_v1..v4 leads to per-node memory exhaustion reachable from any unauthenticated P2P peer (resubmission with runtime ...
* \#75830 \[BC-Medium] SP1 zkVM KZG Point-Evaluation Precompile Silently Accepts Invalid Proofs
* \#75576 \[BC-Medium] Proof executor mishandles \`DestroyedChanged\` accounts, halting proof generation
* \#76092 \[BC-Medium] Missing Decompressed Length Validation in Gossip compute\_message\_id Enables Remote OOM Crash
* \#75721 \[BC-Medium] Engine precompile cache is bounded by entries, not bytes
* \#75616 \[BC-Medium] Base V1 Succinct ZKVM P256 Precompile Override Produces Divergent State and Output Roots
* \#74978 \[BC-Medium] A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
* \#75116 \[SC-Medium] ZK Proof Config Binding Bypass Allows Proofs for Base Sepolia to Execute Under a Forged Rollup Config
* \#75655 \[BC-Medium] Base Azul P256 precompile mismatch makes valid blocks unprovable by OP-Succinct
* \#76203 \[BC-Medium] Holocene \`BatchStream\` consumes and drops \`Undecided\` span batches
* \#74849 \[BC-Medium] P2P Gossip Flood Limiter Bypass via Invalid-Signature Block Spam
* \#75096 \[BC-Medium] \`da\_backlog\_bytes()\` incorrectly reports the backlog
* \#74652 \[BC-Medium] Missing cache invalidation in base-challenger GameScanner leads to fleet-wide circumvention of the dispute-challenge mechanism after governance setImplementation
* \#74913 \[BC-Medium] Unbounded Snappy decompression in libp2p gossip handling enables amplification-class resource exhaustion of base-consensus follower fleet
* \#76540 \[BC-Medium] WitnessExecutor proves root without proving claimed L2 block number was reached
* \#75627 \[BC-Medium] Incorrect Azul ZK precompile overrides can make valid Base proofs attest to non-canonical withdrawal state
* \#76321 \[SC-Medium] Supplemental Technical Correction for Report #76106: Improper L1-Head Binding in AggregateVerifier Initial Proof Verification Can Finalize an Invalid State Root on L1
* \#76166 \[BC-Medium] Post-Isthmus \`withdrawals\_root\` validation is effectively dead code due to fail-open on parent state lookup
* \#75230 \[BC-Medium] ZK prover service hardcodes DEFAULT\_INTERMEDIATE\_ROOT\_INTERVAL=10 while on-chain INTERMEDIATE\_BLOCK\_INTERVAL=30, causing journal digest mismatch that blocks all ZK challenge proofs
* \#76147 \[BC-Medium] Holocene Deposit-Only Fallback Accepts Incomplete-Witness Errors — Prover Can Erase User Transactions From Proven State
* \#75829 \[BC-Medium] Challenger retries for failed ZK proof jobs reuse a terminal session id and do not requeue proving
* \#75347 \[BC-Medium] ZK proof program uses a non-canonical precompile table, breaking soundness across every Base hardfork and enabling permanent chain split
* \#74777 \[BC-Medium] Base node remote Denial of Service
* \#75615 \[BC-Medium] Isthmus withdrawals\_root check is skipped for in-memory parents
* \#74772 \[BC-Medium] Remote Unauthenticated EL Crash via \`admin\_postUnsafePayload\` and Zero-Elasticity Base-Fee
* \#75034 \[BC-Medium] Unbounded Brotli decompression in Flashblock message decoding leads to node memory/CPU exhaustion (DoS)
* \#75107 \[BC-Medium] Stateless trie drops accounts changed after selfdestruct
* \#75256 \[BC-Medium] \`base-proposer\` uses the wrong parent after ASR advancement plus full recovery
* \#75317 \[BC-Medium] FPVM recomputes blob base fee from Jovian DA footprint

</details>

<details>

<summary>Low</summary>

* \#74742 \[BC-Low] Blob-mode frame size mismatch can stall L1 blob publication before submission
* \#75939 \[BC-Low] Unbounded Storage-Key Array in \`eth\_getProof\` Enables Unauthenticated DoS
* \#74494 \[SC-Low] base-prover-zk drops challenger-supplied l1\_head before worker proof request
* \#74495 \[SC-Low] Missing proof threshold check in AggregateVerifier resolution scheduling leads to permanent freezing of dispute game bonds
* \#74351 \[SC-Low] Bond Permanently Locked After ZK Verifier Nullification in PROOF\_THRESHOLD=2 Game
* \#76505 \[BC-Low] Proposer JIT Freshness Check Reuses Cached Result
* \#76521 \[BC-Low] Ignored inbound WebSocket frames can OOM the websocket proxy
* \#74385 \[SC-Low] Bond permanently locked when PROOF\_THRESHOLD == 2 and the complementary verifier is globally nullified mid-game
* \#76037 \[BC-Low] Unbounded \`tokio::spawn\` on Discv5 Driver Events Allows Sustained CPU/Heap DoS Reachable Without a libp2p Peer Connection
* \#76483 \[BC-Low] External timeout around non-cancellation-safe TxManager send can leak the proposer's nonce and stall L1 proposal submission
* \#74546 \[SC-Low] Permanent bond lockup when \`PROOF\_THRESHOLD = 2\` after \`challenge()\` + \`nullify()\` sequence
* \#76299 \[BC-Low] \`eth\_getProof\` accepts an unbounded \`Vec\<JsonStorageKey>\`, letting a single public RPC request walk the state trie tens of thousands of times and amplify response payload by \~100x
* \#76311 \[BC-Low] Valid late metering is discarded after a non-\`MeteringDataPending\` non-inclusion, allowing Enforce-mode per-transaction and per-flashblock resource budgets to be bypassed
* \#76538 \[BC-Low] Post-execution max\_gas\_per\_txn rejection retries over-cap transactions for free
* \#75698 \[BC-Low] Missing Same-Variant FinalizeTask Ordering Allows Finalized Head Regression
* \#75570 \[SC-Low] PR-2372 — \`try\_anchor\_update\` retries reverting \`setAnchorState\` every poll tick, gas-burning the challenger and stalling the bond pipeline
* \#75330 \[BC-Low] Mempool deadline after successful publish can abandon the next required nonce
* \#76368 \[BC-Low] Challenger \`AwaitingProof\` Phase Has No Timeout
* \#75078 \[BC-Low] Clock Domain Conflation in Bond Manager's \`estimate\_unlock\_time\` Delays Bond Claims by Up to 7 Days After Every Process Restart
* \#74367 \[SC-Low] Permanent Bond Locking When PROOF\_THRESHOLD = 2 (Residual Bug After Audit 1 Fix)
* \#76066 \[BC-Low] Base proofs-history eth\_getProof override removes upstream resource guards, allowing low-concurrency RPC requests to inflate node CPU and RSS
* \#76082 \[BC-Low] Checkpoint witness accumulation can trigger OOM in Nitro prover
* \#75376 \[SC-Low] Permanent freezing of dispute game bonds in AggregateVerifier when PROOF\_THRESHOLD equals two and one proof type permanently fails
* \#75378 \[BC-Low] Non-exact EIP-2718 comparison lets safe-head consolidation diverge from exact L1 derivation
* \#74427 \[SC-Low] Permanent Freezing of First-Game Bonds with No Recovery Path + Cascading Liveness Halt on Descendants (PROOF\_THRESHOLD=2)
* \#75146 \[BC-Low] Offchain challenger ReadyToSubmit submit retries are unbounded on deterministic parent-invalid reverts
* \#75433 \[SC-Low] Permanent Bond Lock When PROOF\_THRESHOLD = 2 and Only One Proof Is Submitted in AggregateVerifier
* \#75502 \[SC-Low] Nullified ZK Verifier Does Not Invalidate Prior ZK Games, Allowing Invalid Roots to Finalize After a Soundness Alert
* \#75608 \[SC-Low] Production registrar's \`DEFAULT\_TRUSTED\_CERTS\_PREFIX = 1\` voids \`revokeCert\` on the next routine signer registration; attacker holding the revoked CA key registers a TEE signer that ...
* \#74854 \[SC-Low] Proposer's bond permanently locked after ZK nullification at PROOF\_THRESHOLD=2
* \#74604 \[SC-Low] System Pause Disables Verifier Kill-Switch, Allowing Unchecked Dispute Progression
* \#76546 \[BC-Low] Memory leak in consensus through malicious discv5 peer recommendations because of eager dialing leading to rollup node crash
* \#75479 \[BC-Low] BondManager Retries Premature \`claimCredit()\` Withdrawal on Every Poll After Restart Recovery
* \#75737 \[BC-Low] Unbounded Submit Retry on Invalid Parent Game Freezes Proposer Progress
* \#76287 \[BC-Low] Async batcher L1 transactions permanently skip a dropped nonce after mempool deadline
* \#75878 \[BC-Low] TxPoolRpcExtension Uses merge\_configured Causing Admin Txpool Management Methods to Bypass Module Whitelist on Public RPC Port
* \#75310 \[BC-Low] Offchain proposer submit retries are unbounded on deterministic parent-invalid proposal reverts
* \#75312 \[BC-Low] Offchain challenger nullify retries are unbounded when dual-proof game is externally challenged after proof readiness
* \#76081 \[BC-Low] \`builder.max\_gas\_per\_txn\` over-cap transactions are repeatedly re-executed because post-execution gas-cap rejects are not permanently evicted
* \#76464 \[BC-Low] Transient Metering Cache Poisoning Allows High-Compute Transactions to Bypass Builder Execution Limits
* \#76025 \[BC-Low] Unbounded \`tokio::spawn\` and INFO-Level Logging on \`payload\_by\_number\` Sync Stream — Log/CPU/Memory DoS Reachable from Any Connected Peer
* \#75657 \[BC-Low] \`BatchType::from(u8)\` Panics on Unknown Byte; \`let Ok(...)\` Caller Cannot Intercept → Derivation Halt on Every Node
* \#75880 \[BC-Low] TxPoolRpcExtension exposes admin txpool deletion on standard HTTP/WS transports, enabling targeted pending transaction censorship and >500% confirmation delay
* \#75294 \[BC-Low] The proposer retries an invalid cached parent instead of refreshing recovery state
* \#74600 \[SC-Low] Proof-Threshold Logic Bug Permanently Locks Game Bond
* \#74557 \[SC-Low] Pause Blocks Verifier Nullification But Not Game Resolution
* \#74357 \[SC-Low] Unresolvable Game State When PROOF\_THRESHOLD = 2 and Deadline Is Missed in AggregateVerifier
* \#74945 \[SC-Low] Permanent bond lockup when verifier nullified with PROOF\_THRESHOLD=2
* \#74396 \[SC-Low] Bond locked when game timer expires with insufficient proofs due to conditional fallback gap in claimCredit
* \#74702 \[SC-Low] Game Deadlock And Frozen Bond When PROOF\_THRESHOLD == 2
* \#74911 \[SC-Low] AggregateVerifier bond permanently locked when PROOF\_THRESHOLD=2 and a verifier is nullified — resolve() and claimCredit() both revert
* \#74751 \[SC-Low] Stale Proofs in Other Games Remain Decisive and Can No Longer Be Nullified After Verifier Nullification
* \#75333 \[BC-Low] Off-by-one in batcher default frame size causes permanent blob encoding failure and synchronous livelock halting L2 finalization
* \#75296 \[SC-Low] Permanent freezing of dispute game bonds in \`AggregateVerifier\` on dual proof chains(\`PROOF\_THRESHOLD >= 2\`) due to unreachable \`claimCredit\` fallback
* \#75108 \[SC-Low] Permanent freezing of proposer bond when PROOF\_THRESHOLD=2 and a challenged TEE proof's ZK challenge gets nullified in AggregateVerifier
* \#75488 \[BC-Low] Off-by-one in batcher default frame size causes permanent livelock in \`submit\_pending()\`, halting L2 finalization and freezing all pending withdrawals

</details>

<details>

<summary>Insight</summary>

* \#76173 \[BC-Insight] Default admin RPC exposure lets a malformed unsafe payload reach a critical consensus error
* \#76176 \[BC-Insight] # Missing Parent Hash Verification in Cached Execution Provider — Reorg Serves Stale State from Wrong Fork
* \#74891 \[BC-Insight] Txpool admission omits operator-fee solvency
* \#76296 \[BC-Insight] \[Critical] Kona: Three Derivation Divergence Bugs Cause TEE to Attest Wrong Output Root - Single-Proof Bridge Drain
* \#76301 \[BC-Insight] Base Flashblocks eth\_simulateV1 pending-state expansion enables remote node DoS
* \#74664 \[BC-Insight] Unauthenticated \`admin\_postUnsafePayload\` RPC chains with silent Isthmus \`withdrawals\_root\` validator bypass to poison any Base consensus node's unsafe head and stall finality
* \#75032 \[BC-Insight] \`--rpc.enable-admin=false\` is a dead flag — admin namespace registered regardless on every \`base-consensus\` binary
* \#74725 \[BC-Insight] Post-Isthmus Txpool Admission Omits Operator-Fee Affordability, Causing Repeated Processing of Unexecutable Mempool Transactions
* \#76261 \[BC-Insight] Initial reset uses L2 safe-head timestamp for Granite \`channel\_timeout\` gate, returning the wrong \`SystemConfig\`
* \#75432 \[BC-Insight] Attacker can force Unsafe-Head adoption through \`admin\_postUnsafePayload\` even when admin RPC is not enabled
* \#75630 \[BC-Insight] Race condition in \`OpSuccinctBackend::process\_proof\_request\` causes N-fold duplicate SNARK Groth16 jobs per ZK proof request
* \#75268 \[BC-Insight] \`rpc.enable-admin = false\` Does Not Disable Consensus \`admin\_\*\` RPC Methods
* \#76279 \[BC-Insight] Dead-code \`--rpc.enable-admin\` gate exposes admin RPC namespace on every base-consensus node, allowing unauthenticated \`admin\_postUnsafePayload\` to bypass sequencer signature v...
* \#74653 \[BC-Insight] base-consensus exposes admin RPC despite --rpc.enable-admin being disabled, enabling hidden unsafe-branch partition via admin\_postUnsafePayload
* \#75535 \[BC-Insight] MPT trie node decoder panics on empty leaf or extension path, breaking fault proof liveness
* \#75422 \[BC-Insight] # Quadratic O(N×M) Algorithmic Regression in \`SpanBatch::get\_singular\_batches\` Causes 120× Slowdown in Derivation
* \#76399 \[BC-Insight] TOCTOU Race in SNARK Session Creation Allows Duplicate SP1 Cluster Job Submission, Leading to Permanent Proof Loss
* \#74638 \[BC-Insight] Missing L1OriginTooOld handler in proposer causes infinite retry loop, halting L2 finality and L2→L1 withdrawals
* \#75398 \[BC-Insight] P2P block validation performs expensive operations before cheap signature check
* \#74580 \[BC-Insight] WebSocket Proxy Trusts Spoofed Forwarded IP For Rate Limits
* \#75148 \[BC-Insight] Fjord derivation rejects valid Brotli channels at the activation boundary
* \#75886 \[BC-Insight] Consensus RPC Registers Admin Methods Even When Admin API Is Disabled
* \#76151 \[BC-Insight] eth\_simulateV1(pending) clones Flashblocks pending state per simulated block before request-limit enforcement, delaying eth\_sendRawTransaction and local confirmation beyond 500%
* \#76248 \[BC-Insight] Holocene channel compressed-size cap uses estimated memory size, not raw compressed size
* \#76179 \[BC-Insight] # Cumulative L1 Data Fee Not Tracked in Mempool Sender Balance — 1111× Transaction Overaccept
* \#75597 \[BC-Insight] Lossy ingress metering fanout lets later lower-fee transactions outrank earlier higher-fee transactions under builder metering wait mode
* \#74371 \[SC-Insight] ZKVerifier.sol declares SP1VerificationFailed error that is never raised, leaving the SP1 trust boundary without a defensive wrapper
* \#75919 \[SC-Insight] InvalidProofThreshold Error Declared But Never Thrown in AggregateVerifier
* \#74423 \[SC-Insight] Boundary-valid Nitro attestations are rejected by timestamp validation
* \#76262 \[BC-Insight] P2P Peer Ban Configuration Is Dropped Before Runtime, Preventing Low-Score Peer Disconnection
* \#75734 \[BC-Insight] Base V1 Payload Builder Silently Skips Invalid Derived Transactions, Allowing Post-Error State Mutation
* \#76031 \[BC-Insight] Privileged admin and P2P RPC are publicly exposed

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#76155 \[SC-Medium] Proof Witness Holocene Fallback Can Relabel Witness Errors Into Deposit-Only State Roots
* \#74494 \[SC-Low] base-prover-zk drops challenger-supplied l1\_head before worker proof request
* \#74495 \[SC-Low] Missing proof threshold check in AggregateVerifier resolution scheduling leads to permanent freezing of dispute game bonds
* \#74351 \[SC-Low] Bond Permanently Locked After ZK Verifier Nullification in PROOF\_THRESHOLD=2 Game
* \#74385 \[SC-Low] Bond permanently locked when PROOF\_THRESHOLD == 2 and the complementary verifier is globally nullified mid-game
* \#74546 \[SC-Low] Permanent bond lockup when \`PROOF\_THRESHOLD = 2\` after \`challenge()\` + \`nullify()\` sequence
* \#76065 \[SC-Medium] EndOfSource handling lets a ZK range proof claim an unreached L2 block
* \#75570 \[SC-Low] PR-2372 — \`try\_anchor\_update\` retries reverting \`setAnchorState\` every poll tick, gas-burning the challenger and stalling the bond pipeline
* \#74367 \[SC-Low] Permanent Bond Locking When PROOF\_THRESHOLD = 2 (Residual Bug After Audit 1 Fix)
* \#75113 \[SC-Medium] ZK Challenge Proofs Use 10-Block Intermediate Roots While AggregateVerifier Verifies 30-Block Segments, Preventing ZK Correction of Invalid Proposals
* \#75376 \[SC-Low] Permanent freezing of dispute game bonds in AggregateVerifier when PROOF\_THRESHOLD equals two and one proof type permanently fails
* \#74427 \[SC-Low] Permanent Freezing of First-Game Bonds with No Recovery Path + Cascading Liveness Halt on Descendants (PROOF\_THRESHOLD=2)
* \#75433 \[SC-Low] Permanent Bond Lock When PROOF\_THRESHOLD = 2 and Only One Proof Is Submitted in AggregateVerifier
* \#75502 \[SC-Low] Nullified ZK Verifier Does Not Invalidate Prior ZK Games, Allowing Invalid Roots to Finalize After a Soundness Alert
* \#75608 \[SC-Low] Production registrar's \`DEFAULT\_TRUSTED\_CERTS\_PREFIX = 1\` voids \`revokeCert\` on the next routine signer registration; attacker holding the revoked CA key registers a TEE signer that ...
* \#75423 \[SC-Medium] ZK Range Client Accepts Truncated Post-Azul Execution, Allowing Invalid AggregateVerifier State Roots
* \#75626 \[SC-Medium] rc28 OP-Succinct EndOfSource relabels an early output root as a later AggregateVerifier sequence
* \#74854 \[SC-Low] Proposer's bond permanently locked after ZK nullification at PROOF\_THRESHOLD=2
* \#74604 \[SC-Low] System Pause Disables Verifier Kill-Switch, Allowing Unchecked Dispute Progression
* \#75249 \[SC-Medium] rc.28 ZK interval mismatch prevents dual-proof fast finality
* \#75686 \[SC-Medium] L1 Source Exhaustion Lets Range Proofs Finalize a Future L2 Block With an Old Output Root
* \#75413 \[SC-High] ZK Proof Executor Derives BLOBBASEFEE From Base's DA-Footprint Header Field, Allowing Invalid Output Roots
* \#75116 \[SC-Medium] ZK Proof Config Binding Bypass Allows Proofs for Base Sepolia to Execute Under a Forged Rollup Config
* \#74600 \[SC-Low] Proof-Threshold Logic Bug Permanently Locks Game Bond
* \#74557 \[SC-Low] Pause Blocks Verifier Nullification But Not Game Resolution
* \#74357 \[SC-Low] Unresolvable Game State When PROOF\_THRESHOLD = 2 and Deadline Is Missed in AggregateVerifier
* \#74945 \[SC-Low] Permanent bond lockup when verifier nullified with PROOF\_THRESHOLD=2
* \#74396 \[SC-Low] Bond locked when game timer expires with insufficient proofs due to conditional fallback gap in claimCredit
* \#74702 \[SC-Low] Game Deadlock And Frozen Bond When PROOF\_THRESHOLD == 2
* \#74911 \[SC-Low] AggregateVerifier bond permanently locked when PROOF\_THRESHOLD=2 and a verifier is nullified — resolve() and claimCredit() both revert
* \#74751 \[SC-Low] Stale Proofs in Other Games Remain Decisive and Can No Longer Be Nullified After Verifier Nullification
* \#76321 \[SC-Medium] Supplemental Technical Correction for Report #76106: Improper L1-Head Binding in AggregateVerifier Initial Proof Verification Can Finalize an Invalid State Root on L1
* \#75296 \[SC-Low] Permanent freezing of dispute game bonds in \`AggregateVerifier\` on dual proof chains(\`PROOF\_THRESHOLD >= 2\`) due to unreachable \`claimCredit\` fallback
* \#75108 \[SC-Low] Permanent freezing of proposer bond when PROOF\_THRESHOLD=2 and a challenged TEE proof's ZK challenge gets nullified in AggregateVerifier
* \#74371 \[SC-Insight] ZKVerifier.sol declares SP1VerificationFailed error that is never raised, leaving the SP1 trust boundary without a defensive wrapper
* \#75919 \[SC-Insight] InvalidProofThreshold Error Declared But Never Thrown in AggregateVerifier
* \#74423 \[SC-Insight] Boundary-valid Nitro attestations are rejected by timestamp validation

</details>

<details>

<summary>Blockchain/DLT</summary>

* \#76156 \[BC-Medium] Trailing DER alias bypasses Nitro certificate revocation
* \#74742 \[BC-Low] Blob-mode frame size mismatch can stall L1 blob publication before submission
* \#76173 \[BC-Insight] Default admin RPC exposure lets a malformed unsafe payload reach a critical consensus error
* \#76176 \[BC-Insight] # Missing Parent Hash Verification in Cached Execution Provider — Reorg Serves Stale State from Wrong Fork
* \#76180 \[BC-Medium] Late frame confirmations can prune timed-out channels
* \#74891 \[BC-Insight] Txpool admission omits operator-fee solvency
* \#74944 \[BC-High] Flashblocks cached execution accepts skipped transactions — engine-level consensus split
* \#75950 \[BC-Medium] DestroyedChanged accounts make proof execution diverge from canonical L2 state or halt
* \#75935 \[BC-Medium] Unbounded Snappy Decompression in Gossip Message ID Computation Causes Pre-Validation Consensus Node Resource Exhaustion
* \#75939 \[BC-Low] Unbounded Storage-Key Array in \`eth\_getProof\` Enables Unauthenticated DoS
* \#75938 \[BC-High] Span batch encoder omits genesis timestamp, causing all span batches to decode as future batches and be dropped post-Holocene
* \#76136 \[BC-Medium] \`OnlineHostBackend::get\_preimage\` Contains an Unconditional Infinite Retry Loop
* \#74480 \[BC-Critical] Gossip snappy decompression missing MAX\_GOSSIP\_SIZE check lets any peer DoS base-consensus
* \#74760 \[BC-Medium] State-root derivation divergence between base-reth-node and base-proof-executor lets unprivileged BLOBBASEFEE transactions stall TEE finality for affected L2 ranges
* \#76497 \[BC-High] Detached Task Semaphore Permit Leak in Consensus-Layer RPC Processor Starves EngineActor Router and Halts Block Production
* \#76505 \[BC-Low] Proposer JIT Freshness Check Reuses Cached Result
* \#76500 \[BC-High] Batcher Reorg Reset Skips Canonical Blocks Via Stale Safe Head
* \#76521 \[BC-Low] Ignored inbound WebSocket frames can OOM the websocket proxy
* \#76527 \[BC-High] Unauthenticated \`optimism\_outputAtBlock\` flood halts sequencer block production via EngineActor head-of-line blocking
* \#74531 \[BC-Critical] Gossipsub message-id computation Snappy-decompresses untrusted payloads without a decompressed-size cap
* \#74823 \[BC-Medium] min\_base\_fee - Consolidation bypass, below-floor tx inclusion + proof divergence
* \#74391 \[BC-Critical] Missing Snappy Decoded-Length Bounds in CL Gossip Enable Batched Pre-Validation CPU / Allocator Churn
* \#76037 \[BC-Low] Unbounded \`tokio::spawn\` on Discv5 Driver Events Allows Sustained CPU/Heap DoS Reachable Without a libp2p Peer Connection
* \#76045 \[BC-Medium] Malformed batch will be retried as invalid payload attributes forever
* \#76483 \[BC-Low] External timeout around non-cancellation-safe TxManager send can leak the proposer's nonce and stall L1 proposal submission
* \#74791 \[BC-Medium] Stale \`intermediate\_block\_interval\` Cache Forces Dispute Games Into Incorrect Resolved State
* \#76470 \[BC-Medium] Short ZK range proof can be accepted as a full AggregateVerifier interval and trigger global ZK verifier nullification
* \#75972 \[BC-Medium] ZK range proofs can stop at EndOfSource but still commit the requested target block, allowing a short-range proof to finalize an invalid longer-range output root
* \#75554 \[BC-Critical] Unbounded snappy decompression in NetworkPayloadEnvelope::decode\_v1..v4 leads to per-node memory exhaustion reachable from any unauthenticated P2P peer
* \#74673 \[BC-Medium] Proof executor uses non-canonical BLOBBASEFEE after Jovian/Base V1
* \#76386 \[BC-High] Time-controlled \`DisputeGameFactory.create()\` pins \`game.l1Head()\` to a batch-incomplete L1 block, making early-halt finalization on-chain unchallengeable
* \#76282 \[BC-High] EngineActor Single-Threaded Routing Blocked by RPC Flood Starves All Consensus Operations
* \#76296 \[BC-Insight] \[Critical] Kona: Three Derivation Divergence Bugs Cause TEE to Attest Wrong Output Root - Single-Proof Bridge Drain
* \#76299 \[BC-Low] \`eth\_getProof\` accepts an unbounded \`Vec\<JsonStorageKey>\`, letting a single public RPC request walk the state trie tens of thousands of times and amplify response payload by \~100x
* \#76301 \[BC-Insight] Base Flashblocks eth\_simulateV1 pending-state expansion enables remote node DoS
* \#76311 \[BC-Low] Valid late metering is discarded after a non-\`MeteringDataPending\` non-inclusion, allowing Enforce-mode per-transaction and per-flashblock resource budgets to be bypassed
* \#76538 \[BC-Low] Post-execution max\_gas\_per\_txn rejection retries over-cap transactions for free
* \#75902 \[BC-Medium] Access Control Bypass in Consensus RPC 'admin\_postUnsafePayload' Can Lead to Chain Split via Skipped Isthmus Withdrawals-Root Validation
* \#76222 \[BC-Medium] Isthmus \`withdrawals\_root\` validation is skipped when the parent is only in the engine-tree overlay, allowing invalid multi-block unsafe/sync targets to be accepted
* \#74657 \[BC-Critical] Remote Node DoS via Unbounded Snappy Decompression in Gossip Message Processing
* \#74664 \[BC-Insight] Unauthenticated \`admin\_postUnsafePayload\` RPC chains with silent Isthmus \`withdrawals\_root\` validator bypass to poison any Base consensus node's unsafe head and stall finality
* \#74678 \[BC-Medium] Missing \`requests\_hash\` Validation in the Raw Import Path Can Persist Invalid Post-Isthmus Blocks and Cause Chain Split
* \#75698 \[BC-Low] Missing Same-Variant FinalizeTask Ordering Allows Finalized Head Regression
* \#75032 \[BC-Insight] \`--rpc.enable-admin=false\` is a dead flag — admin namespace registered regardless on every \`base-consensus\` binary
* \#75623 \[BC-Medium] Post-Holocene span batch prefix validation can allow partially derived batches from a span rejected by full validation
* \#75330 \[BC-Low] Mempool deadline after successful publish can abandon the next required nonce
* \#76232 \[BC-Medium] SP1 ZK proof validation accepts a claimed L2 block number that derivation never reached
* \#76368 \[BC-Low] Challenger \`AwaitingProof\` Phase Has No Timeout
* \#75911 \[BC-Medium] Base consensus gossip validates block payloads before signer verification, enabling remote CPU DoS
* \#75912 \[BC-Medium] Proof executor derives BLOBBASEFEE from Jovian DA footprint while Base nodes hard-code it to 1
* \#75078 \[BC-Low] Clock Domain Conflation in Bond Manager's \`estimate\_unlock\_time\` Delays Bond Claims by Up to 7 Days After Every Process Restart
* \#74725 \[BC-Insight] Post-Isthmus Txpool Admission Omits Operator-Fee Affordability, Causing Repeated Processing of Unexecutable Mempool Transactions
* \#75354 \[BC-Medium] Hardcoded 10-block ZK witness interval makes 30-block challenge proofs unverifiable
* \#75709 \[BC-Medium] ZK range proofs can commit a later L2 block number while only deriving an earlier safe head, allowing stale output roots to be proven for the wrong L2 block
* \#76261 \[BC-Insight] Initial reset uses L2 safe-head timestamp for Granite \`channel\_timeout\` gate, returning the wrong \`SystemConfig\`
* \#75363 \[BC-Medium] ZK proof program uses a non-canonical precompile table, breaking soundness across every Base hardfork and enabling permanent chain split
* \#75710 \[BC-High] Cross-topic MessageId-cache poisoning blackholes block propagation in base-consensus-gossip
* \#76066 \[BC-Low] Base proofs-history eth\_getProof override removes upstream resource guards, allowing low-concurrency RPC requests to inflate node CPU and RSS
* \#75101 \[BC-Critical] ZK Proof Executor Uses Wrong Azul Precompile Semantics, Allowing Proofs For Invalid L2 State Roots
* \#75369 \[BC-Medium] ZK range proof silently truncates derivation on \`EndOfSource\` and commits the original (T, R\_N) — permanent chain split when \`claimed\_l2\_block\_number\` outruns the sequenced L1 bat...
* \#76082 \[BC-Low] Checkpoint witness accumulation can trigger OOM in Nitro prover
* \#74569 \[BC-Medium] Base consensus client process-exits on safe \`INVALID\` payloads because \`is\_deposits\_only\` misclassifies every derived block and bypasses Holocene fallback
* \#75378 \[BC-Low] Non-exact EIP-2718 comparison lets safe-head consolidation diverge from exact L1 derivation
* \#76410 \[BC-Medium] Stateless \`GameScanner\` permanently drops invalid in-progress dispute games whose factory index falls below \`gameCount - lookback\_games\` after any single missed tick
* \#75146 \[BC-Low] Offchain challenger ReadyToSubmit submit retries are unbounded on deterministic parent-invalid reverts
* \#75390 \[BC-Medium] Invalid Derived Sequencer Txs Make EL Safe Roots Unprovable
* \#75305 \[BC-Critical] Unbounded Memory Allocation in Gossip Message Processing Allows Unauthenticated Attacker to Freeze Block Propagation
* \#76107 \[BC-Medium] Gossip block validation performs the expensive payload clone, RLP re-encode and keccak before checking the cheap unsafe block signer signature, enabling sustained CPU and heap amp...
* \#76113 \[BC-Medium] Base ZK range proof can attest a 600-block checkpoint using only a partially derived EndOfSource range
* \#76425 \[BC-Medium] \`eth\_getProof\` response is not bound to \`L2ToL1MessagePasser\`, allowing a substituted bridge storage root to be accepted
* \#75979 \[BC-Medium] \`compute\_message\_id\` decompresses every incoming gossip message via \`snap::raw::Decoder::decompress\_vec\` without enforcing the documented decompressed-size cap, allowing per-messa...
* \#75410 \[BC-High] Cross-topic message ID collision in gossipsub allows attacker to censor blocks from network nodes
* \#75986 \[BC-Medium] Proposer cold recovery loses the current anchor game address after ASR advances, causing subsequent multiproof games to revert
* \#75992 \[BC-Medium] Cold-restart bootstrap break every honest proposer is permanently bricked after the first anchor advance, freezing L2→L1 withdrawals
* \#75719 \[BC-Medium] Proof executor unconditionally deletes destroyed-and-recreated accounts, producing invalid state roots that block L1 finalization
* \#75849 \[BC-Medium] Late challenge transitions can evade challenger scanning after aging out of the lookback window
* \#75432 \[BC-Insight] Attacker can force Unsafe-Head adoption through \`admin\_postUnsafePayload\` even when admin RPC is not enabled
* \#76454 \[BC-Medium] Range program \`EndOfSource\` guard dead, enabling trivial-proof finalization halt
* \#75437 \[BC-Medium] Jovian min\_base\_fee Omission Lets Consolidation Promote a Non-Canonical Unsafe Block as Safe
* \#75630 \[BC-Insight] Race condition in \`OpSuccinctBackend::process\_proof\_request\` causes N-fold duplicate SNARK Groth16 jobs per ZK proof request
* \#75220 \[BC-Medium] Isthmus withdrawalsRoot validation skipped when parent state unavailable, allowing invalid blocks to pass post-execution validation
* \#74856 \[BC-Medium] Replay of stale sequencer-signed unsafe gossip rewinds unsafe\_head and causes unintended L2 reorg behavior
* \#75905 \[BC-Medium] Permissionless Bridge Withdrawal Halt via \`BLOBBASEFEE\` Divergence Between Reth and the Proof Program
* \#76536 \[BC-Medium] \[Medium]\[Blockchain/DLT] Proposer cold-start uses ASR sentinel as parent after anchor advancement, causing AggregateVerifier.initializeWithInitData to revert UnexpectedBlockNumber...
* \#76546 \[BC-Low] Memory leak in consensus through malicious discv5 peer recommendations because of eager dialing leading to rollup node crash
* \#75469 \[BC-Critical] Gossip payload decoder allocates unbounded Snappy output
* \#75479 \[BC-Low] BondManager Retries Premature \`claimCredit()\` Withdrawal on Every Poll After Restart Recovery
* \#75908 \[BC-Medium] ZK range proof does not bind the actual derived end block or intermediate-root cadence, allowing a shorter range to advance ASR as a full interval
* \#76075 \[BC-Medium] Proof executor unconditionally deletes destroyed-and-recreated accounts, producing invalid state roots that block L1 finalization
* \#75268 \[BC-Insight] \`rpc.enable-admin = false\` Does Not Disable Consensus \`admin\_\*\` RPC Methods
* \#75485 \[BC-Critical] Unbounded Snappy Decompression in GossipSub \`message\_id\_fn\` Causes Per-Message Memory Spike of \~428 MiB Before Validation
* \#75737 \[BC-Low] Unbounded Submit Retry on Invalid Parent Game Freezes Proposer Progress
* \#74864 \[BC-Medium] Isthmus Withdrawals Root Validation Bypass Leading to Invalid Block Acceptance
* \#76294 \[BC-Medium] Isthmus withdrawals\_root validator silently accepts malformed blocks, producing peer divergence between honest Base nodes processing identical sequencer payloads (chain-level fork)
* \#75013 \[BC-Medium] DA backlog accounting misses encoded unpublished data, causing mempool processing beyond configured throttle limits
* \#75814 \[BC-Medium] Retryable span validation discards valid L1 batches and allows empty safe derivation
* \#74656 \[BC-Critical] Gossip message\_id\_fn performs uncapped Snappy decompression before deduplication, enabling P2P memory-amplification DoS
* \#75031 \[BC-Medium] Default blob frame sizing can create unencodable frames, causing retry storms and >30% batcher resource amplification
* \#76287 \[BC-Low] Async batcher L1 transactions permanently skip a dropped nonce after mempool deadline
* \#75288 \[BC-Medium] ProofsStorage silently drops newly written storage slots for destroyed-and-recreated accounts, breaking proof generation
* \#76279 \[BC-Insight] Dead-code \`--rpc.enable-admin\` gate exposes admin RPC namespace on every base-consensus node, allowing unauthenticated \`admin\_postUnsafePayload\` to bypass sequencer signature v...
* \#76404 \[BC-Medium] Stateless challenger lookback window allows unbounded eviction of pending invalid proposals via DisputeGameFactory.create() spam
* \#75673 \[BC-Medium] Base V1 ZK proofs execute the old P256 precompile, allowing valid SP1 proofs for invalid output roots
* \#75301 \[BC-Critical] Attacker can DoS Consensus gossip via unbounded snappy decompression in message ID computation
* \#75878 \[BC-Low] TxPoolRpcExtension Uses merge\_configured Causing Admin Txpool Management Methods to Bypass Module Whitelist on Public RPC Port
* \#75683 \[BC-High] ProofsStorage silently drops newly written storage slots for destroyed-and-recreated accounts, breaking proof generation
* \#74653 \[BC-Insight] base-consensus exposes admin RPC despite --rpc.enable-admin being disabled, enabling hidden unsafe-branch partition via admin\_postUnsafePayload
* \#75527 \[BC-Medium] ProofsStorage silently drops newly written storage slots for destroyed-and-recreated accounts, breaking proof generation
* \#75826 \[BC-High] Off-by-one in batcher frame sizing causes infinite blob-encode hot loop, halting all L1 publication
* \#75757 \[BC-Medium] Unbounded snappy decompression in NetworkPayloadEnvelope::decode\_v1..v4 leads to per-node memory exhaustion reachable from any unauthenticated P2P peer (resubmission with runtime ...
* \#75533 \[BC-High] SP1 prover precompile map drops Jovian cap and Osaka pricing, enabling a forged \`outputRoot\` and ≥10% bridge drain wherever the ZK fast path is active
* \#75310 \[BC-Low] Offchain proposer submit retries are unbounded on deterministic parent-invalid proposal reverts
* \#75535 \[BC-Insight] MPT trie node decoder panics on empty leaf or extension path, breaking fault proof liveness
* \#75830 \[BC-Medium] SP1 zkVM KZG Point-Evaluation Precompile Silently Accepts Invalid Proofs
* \#75312 \[BC-Low] Offchain challenger nullify retries are unbounded when dual-proof game is externally challenged after proof readiness
* \#76081 \[BC-Low] \`builder.max\_gas\_per\_txn\` over-cap transactions are repeatedly re-executed because post-execution gas-cap rejects are not permanently evicted
* \#76271 \[BC-High] Jovian minBaseFee is ignored during unsafe block consolidation, allowing divergent L2 headers
* \#76464 \[BC-Low] Transient Metering Cache Poisoning Allows High-Compute Transactions to Bypass Builder Execution Limits
* \#75576 \[BC-Medium] Proof executor mishandles \`DestroyedChanged\` accounts, halting proof generation
* \#76092 \[BC-Medium] Missing Decompressed Length Validation in Gossip compute\_message\_id Enables Remote OOM Crash
* \#75721 \[BC-Medium] Engine precompile cache is bounded by entries, not bytes
* \#75616 \[BC-Medium] Base V1 Succinct ZKVM P256 Precompile Override Produces Divergent State and Output Roots
* \#75422 \[BC-Insight] # Quadratic O(N×M) Algorithmic Regression in \`SpanBatch::get\_singular\_batches\` Causes 120× Slowdown in Derivation
* \#76399 \[BC-Insight] TOCTOU Race in SNARK Session Creation Allows Duplicate SP1 Cluster Job Submission, Leading to Permanent Proof Loss
* \#74638 \[BC-Insight] Missing L1OriginTooOld handler in proposer causes infinite retry loop, halting L2 finality and L2→L1 withdrawals
* \#74978 \[BC-Medium] A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
* \#76025 \[BC-Low] Unbounded \`tokio::spawn\` and INFO-Level Logging on \`payload\_by\_number\` Sync Stream — Log/CPU/Memory DoS Reachable from Any Connected Peer
* \#76229 \[BC-High] \`AttributesMatch::check\_eip1559\` Discards \`min\_base\_fee\` from Jovian Block Extra Data — Allows Network Processing Nodes to Process Mempool Transactions Beyond Canonical Set Parameters
* \#74513 \[BC-Critical] Unbounded snappy decompression in gossipsub message\_id\_fn causes pre-validation CPU and memory exhaustion from a mesh peer
* \#75371 \[BC-High] unlimited p2p connections lead to node isolation and RPC crash
* \#75657 \[BC-Low] \`BatchType::from(u8)\` Panics on Unknown Byte; \`let Ok(...)\` Caller Cannot Intercept → Derivation Halt on Every Node
* \#75655 \[BC-Medium] Base Azul P256 precompile mismatch makes valid blocks unprovable by OP-Succinct
* \#75398 \[BC-Insight] P2P block validation performs expensive operations before cheap signature check
* \#75880 \[BC-Low] TxPoolRpcExtension exposes admin txpool deletion on standard HTTP/WS transports, enabling targeted pending transaction censorship and >500% confirmation delay
* \#75294 \[BC-Low] The proposer retries an invalid cached parent instead of refreshing recovery state
* \#76203 \[BC-Medium] Holocene \`BatchStream\` consumes and drops \`Undecided\` span batches
* \#76494 \[BC-High] Batcher \`submit\_pending\` spin loop halts all L1 batch submission
* \#74580 \[BC-Insight] WebSocket Proxy Trusts Spoofed Forwarded IP For Rate Limits
* \#74849 \[BC-Medium] P2P Gossip Flood Limiter Bypass via Invalid-Signature Block Spam
* \#75096 \[BC-Medium] \`da\_backlog\_bytes()\` incorrectly reports the backlog
* \#75148 \[BC-Insight] Fjord derivation rejects valid Brotli channels at the activation boundary
* \#76096 \[BC-Critical] Unintended permanent chain split due to omitted \`min\_base\_fee\` validation in Jovian EIP-1559 implementation
* \#75886 \[BC-Insight] Consensus RPC Registers Admin Methods Even When Admin API Is Disabled
* \#74652 \[BC-Medium] Missing cache invalidation in base-challenger GameScanner leads to fleet-wide circumvention of the dispute-challenge mechanism after governance setImplementation
* \#74913 \[BC-Medium] Unbounded Snappy decompression in libp2p gossip handling enables amplification-class resource exhaustion of base-consensus follower fleet
* \#75962 \[BC-Critical] Pre-Validation Decompression Bomb in GossipSub Leads to Deterministic OOM (428 MiB/msg)
* \#76540 \[BC-Medium] WitnessExecutor proves root without proving claimed L2 block number was reached
* \#75627 \[BC-Medium] Incorrect Azul ZK precompile overrides can make valid Base proofs attest to non-canonical withdrawal state
* \#75333 \[BC-Low] Off-by-one in batcher default frame size causes permanent blob encoding failure and synchronous livelock halting L2 finalization
* \#76166 \[BC-Medium] Post-Isthmus \`withdrawals\_root\` validation is effectively dead code due to fail-open on parent state lookup
* \#75230 \[BC-Medium] ZK prover service hardcodes DEFAULT\_INTERMEDIATE\_ROOT\_INTERVAL=10 while on-chain INTERMEDIATE\_BLOCK\_INTERVAL=30, causing journal digest mismatch that blocks all ZK challenge proofs
* \#75246 \[BC-Critical] Remote Snappy-compressed P2P block gossip message can force excessive memory and CPU usage in Base Azul nodes
* \#75653 \[BC-Critical] Missing operator fee in txpool validation allows costless pool pollution DoS
* \#75868 \[BC-High] Unbounded Snappy Decompression in Public P2P Gossip Bypasses the 10 MiB Pre-Validation Limit
* \#76147 \[BC-Medium] Holocene Deposit-Only Fallback Accepts Incomplete-Witness Errors — Prover Can Erase User Transactions From Proven State
* \#75829 \[BC-Medium] Challenger retries for failed ZK proof jobs reuse a terminal session id and do not requeue proving
* \#75347 \[BC-Medium] ZK proof program uses a non-canonical precompile table, breaking soundness across every Base hardfork and enabling permanent chain split
* \#74777 \[BC-Medium] Base node remote Denial of Service
* \#75856 \[BC-High] Default blob batcher frame size can emit unencodable packed blob frames and delay L1 DA submission beyond 500%
* \#75615 \[BC-Medium] Isthmus withdrawals\_root check is skipped for in-memory parents
* \#74772 \[BC-Medium] Remote Unauthenticated EL Crash via \`admin\_postUnsafePayload\` and Zero-Elasticity Base-Fee
* \#76151 \[BC-Insight] eth\_simulateV1(pending) clones Flashblocks pending state per simulated block before request-limit enforcement, delaying eth\_sendRawTransaction and local confirmation beyond 500%
* \#76248 \[BC-Insight] Holocene channel compressed-size cap uses estimated memory size, not raw compressed size
* \#74620 \[BC-Critical] Unbounded Snappy decompression in Base gossip message-id computation increases node resource consumption
* \#75034 \[BC-Medium] Unbounded Brotli decompression in Flashblock message decoding leads to node memory/CPU exhaustion (DoS)
* \#75107 \[BC-Medium] Stateless trie drops accounts changed after selfdestruct
* \#75256 \[BC-Medium] \`base-proposer\` uses the wrong parent after ASR advancement plus full recovery
* \#75317 \[BC-Medium] FPVM recomputes blob base fee from Jovian DA footprint
* \#75504 \[BC-Critical] Snappy decompression bomb in gossipsub message handler crashes base nodes
* \#76179 \[BC-Insight] # Cumulative L1 Data Fee Not Tracked in Mempool Sender Balance — 1111× Transaction Overaccept
* \#75597 \[BC-Insight] Lossy ingress metering fanout lets later lower-fee transactions outrank earlier higher-fee transactions under builder metering wait mode
* \#76262 \[BC-Insight] P2P Peer Ban Configuration Is Dropped Before Runtime, Preventing Low-Score Peer Disconnection
* \#75734 \[BC-Insight] Base V1 Payload Builder Silently Skips Invalid Derived Transactions, Allowing Post-Error State Mutation
* \#76031 \[BC-Insight] Privileged admin and P2P RPC are publicly exposed
* \#75488 \[BC-Low] Off-by-one in batcher default frame size causes permanent livelock in \`submit\_pending()\`, halting L2 finalization and freezing all pending withdrawals
* \#74556 \[BC-Critical] # Base Azul CL: unbounded snappy decompression in gossipsub \`message\_id\_fn\` allows a single unauthenticated P2P message to OOM-kill every reachable consensus node, halting the c...

</details>
