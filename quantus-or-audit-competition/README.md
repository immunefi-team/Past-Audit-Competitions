# Quantus | Audit Competition

## Reports by Severity

<details>

<summary>Critical</summary>

* \#90207 \[BC-Critical] Missing ZK-tree leaf on the wormhole miner-fee credit leads to permanent freezing of the block author's volume fee
* \#88609 \[BC-Critical] High-security whitelist is incomplete: the transaction tip bypasses the guardian delay entirely
* \#89347 \[BC-Critical] High-security accounts can be drained immediately via an unbounded tip on a whitelisted call
* \#89999 \[W\&A-Critical] Multisig discovery accepts an indexer-supplied address without verifying its derivation
* \#88628 \[BC-Critical] `Balances::transfer_allow_death` debits the sender, credits the recipient nothing, and reports success
* \#89438 \[BC-Critical] Miner-controlled future timestamps bypass reversible-transfer cancellation windows and cause direct loss of funds
* \#88841 \[BC-Critical] High-security accounts are drained irreversibly through the transaction tip, which no high-security check in the runtime models: the guardian delay, cancel and recover\_funds are...
* \#89819 \[BC-Critical] Missing FIPS 204 domain separation in ML-DSA transaction verification lets a victim's message signature authorize a transfer, enabling direct theft of user funds
* \#89832 \[BC-Critical] Unbounded transaction tips bypass high-security containment and drain compromised accounts
* \#89136 \[BC-Critical] Missing deposit-release calls in the high-security whitelist permanently freeze reserved balances, including an uninvolved third party's recovery deposit
* \#88730 \[W\&A-Critical] Generated `MultiAddress::Index` decodes Unit as Compact, letting one signer hide `transfer_all` in an authoritative 2-of-2 multisig proposal and drain the multisig
* \#89007 \[W\&A-Critical] Accessibility bypass of the wallet lock screen permits an unauthenticated sweep of the entire wormhole balance
* \#89802 \[W\&A-Critical] Missing chain-identity validation in the Quantus SDK signing path lets a malicious RPC endpoint obtain signatures valid on an unintended chain

</details>

<details>

<summary>High</summary>

* \#90460 \[W\&A-High] Quantus POS accepts a replaceable pool transaction as payment before execution
* \#89058 \[BC-High] pallet-multisig decodes an opaque inner call with no depth limit: a 4.8 KB nested blob traps the runtime before the weight guard can fire, so the work is never charged
* \#89733 \[BC-High] Wormhole miner volume fee is credited to a keyless account without a ZK-tree leaf, permanently freezing it
* \#89552 \[BC-High] QPoW miners will permanently freeze the sub-quantum remainder of every block reward because `mint_reward()` credits full planck while `hash_leaf()` only commits \`floor(amount / 10^1...
* \#89657 \[W\&A-High] Validation gap in multisig confirmation allows approval of hidden transfer arguments
* \#90398 \[BC-High] An unauthenticated attacker will force processing nodes to process unsigned Wormhole mempool transactions beyond configured proposer limits.
* \#90400 \[BC-High] Opaque multisig proposals bypass the 256-level extrinsic depth limit and repeatedly trap block-author Wasm without a durable fee or nonce charge
* \#89806 \[W\&A-High] Frozen Vesting decode failure leaves an opaque multisig approval actionable and allows a 999.999 QUAN drain
* \#89040 \[BC-High] Wormhole miner-fee credit records no zk-tree leaf, freezing miner yield
* \#89808 \[W\&A-High] Unit `AccountIndex` consumes zero runtime bytes while the SDK consumes the next field, allowing one canonical approval to display no attacker transfer while transferring 999.999 QU...
* \#90351 \[W\&A-High] The multisig approval screen signs call bytes it could not decode, showing the signer a call name and no parameters
* \#89197 \[BC-High] Missing ZK-leaf recording permanently freezes miner-fee balances credited to keyless Wormhole accounts
* \#88485 \[BC-High] Unverified nullifier public inputs bypass Wormhole semantic txpool deduplication
* \#89984 \[BC-High] Missing ZK-tree registration for miner volume fees permanently freezes credited funds
* \#89987 \[W\&A-High] MultiAddress::Index is decoded at a different width by the SDK and the runtime, letting a QR code display a harmless batch while signing a 1000 QUAN transfer to the attacker
* \#88943 \[BC-High] Residue truncation and two-leaf split in pallet-mining-rewards lead to permanent freezing of unclaimed miner yield
* \#88951 \[BC-High] Batch-level fee rounding permanently strands fees locked by Wormhole leaf proofs
* \#89053 \[BC-High] Wormhole miner volume fees are permanently unclaimable because the credit creates no ZK-tree leaf
* \#89250 \[BC-High] Missing quantum-alignment in the wormhole deposit chokepoint leads to permanent freezing of funds
* \#88938 \[BC-High] Missing ZK-tree leaf record in the wormhole miner volume-fee credit path leads to permanent freezing of funds
* \#89283 \[W\&A-High] Runtime-metadata interpretation conflict in Quantus SDK lets a multisig signer conceal `transfer_all` and drain shared funds
* \#88914 \[W\&A-High] Hide Balances masks executable multisig amounts, enabling deceptive `utility.if_else` approvals
* \#90180 \[BC-High] Wormhole miner volume-fee credits are uncollectible because no proof leaf is recorded
* \#88571 \[BC-High] Missing ZK-tree leaf on the wormhole exit miner-fee credit permanently freezes all miner exit-fee revenue (keyless account, no exit path)
* \#88735 \[BC-High] Lossy Wormhole leaf quantization permanently freezes a portion of every non-aligned mining reward and transaction-fee payout
* \#88300 \[BC-High] Missing Wormhole ZK leaf permanently freezes all miner exit-fee yield
* \#89798 \[W\&A-High] Multisig review hides call parameters and enables a treasury drain
* \#89304 \[BC-High] Wormhole miner volume fee is credited without a ZK leaf and can never be spent
* \#88540 \[BC-High] Wormhole miner volume fees are permanently unclaimable because no ZK leaf is created
* \#90235 \[BC-High] Wormhole fee credits skip record\_transfer: miner and aggregator volume fees are minted to keyless addresses with no zk-tree leaf, permanently freezing them
* \#88250 \[BC-High] One unauthenticated TCP burst permanently terminates a node's entire networking stack, with no recovery short of a process restart
* \#88566 \[BC-High] PR #656 incomplete remediation: bundle-level rounding under-settles per-leaf Wormhole fees
* \#89065 \[BC-High] Sub-fee-exitable wormhole deposits are accepted and recorded but can never be spent or exited, permanently freezing user funds
* \#90329 \[BC-High] pallet\_multisig::propose decodes opaque inner calls without a depth limit, trapping the block producer
* \#88771 \[BC-High] Wormhole miner volume-fee is credited to a key-less address with no ZK-tree leaf, permanently freezing it
* \#89134 \[BC-High] Miner volume-fee credits lack ZK-tree leaves, permanently freezing block-author fees
* \#89171 \[BC-High] Missing pending-connection cap and per-IP limit in litep2p TCP transport causes 30%+ resource increase and 500%+ block delays via zombie peer flood
* \#88297 \[BC-High] Scheduled transfers below the existential deposit strand the sender's funds with no owner-accessible recovery
* \#89348 \[BC-High] Mining reward's zk-tree leaf floors the amount, freezing the remainder
* \#88969 \[BC-High] Unbounded recursive decoding of opaque multisig calls makes block authors repeatedly execute unpaid mempool poison
* \#89095 \[W\&A-High] Android process death defeats recovery-phrase clipboard expiry and exposes the mnemonic to later foreground apps
* \#88348 \[W\&A-High] Exponential synchronous decoding of a 213-byte Utility.if\_else proposal leaves the mobile wallet unable to render a frame or accept input whenever the affected multisig account loads
* \#88498 \[BC-High] Non-quantized mining payouts permanently strand approximately 0.786% of block-author yield
* \#89062 \[BC-High] Remote unauthenticated block-production DoS: fee-free unverifiable wormhole exit proofs force unbounded failed ZK verifications on every block producer (up to the full 9s build budg...
* \#90133 \[BC-High] Mining rewards are minted at raw precision but their ZK leaves commit only `floor(amount / 0.01 QUAN)`, permanently freezing every remainder and making ordinary fee yield wholly unc...
* \#88483 \[W\&A-High] Official Multisig.approve displays a zero-value call and still signs a full vault transfer
* \#89448 \[BC-High] Block authors are paid wormhole volume fees at a keyless address that no exit proof can unlock
* \#89449 \[BC-High] Mining rewards and fee credits mint more than their ZK leaf commits, stranding the difference at a keyless miner address
* \#88851 \[W\&A-High] Exponential `Utility::if_else` call display lets a multisig co-signer persistently freeze the mobile wallet
* \#88404 \[BC-High] Preimage reference over-drop in referenda enactment scheduling silently voids a passed governance decision
* \#88667 \[W\&A-High] Unhandled call-decode failure in the multisig approval sheet leads to direct theft of user funds
* \#89151 \[W\&A-High] Primary-only Utility.if\_else approval summary shows 1 QUAN while a 50 QUAN fallback drains the multisig.
* \#88872 \[BC-High] Missing `record_transfer` on the miner volume-fee credit permanently freezes unclaimed miner yield on a non-ML-DSA-signable QPoW wormhole author
* \#88881 \[BC-High] Wormhole miner volume fee is minted without its ZK-tree leaf, permanently freezing every exit fee.
* \#89008 \[W\&A-High] Fail-open multisig call decoding in the mobile app hides transfer\_all and enables theft of the full multisig balance
* \#88223 \[BC-High] Miner volume fee and aggregator rebate are credited to keyless wormhole addresses with no zk-tree leaf, permanently freezing them (incomplete fix of EQ-QNT-WORMHOLE-F-03)
* \#89009 \[BC-High] Miner's share of the wormhole exit volume fee is credited to a keyless address with no ZK leaf, permanently freezing it
* \#89011 \[W\&A-High] Fail-open runtime-call decoding lets a minority multisig signer obtain approval for full-balance theft
* \#89163 \[BC-High] Miner volume fee is reported as paid but records no ZK leaf, so the credited fee is unrealizable by the miner
* \#89036 \[BC-High] Unbounded unauthenticated PQ handshakes permanently terminate node P2P networking and can halt network-wide transaction confirmation
* \#88912 \[BC-High] Missing `record_transfer` in Wormhole miner-fee settlement permanently freezes unclaimed yield
* \#89247 \[BC-High] Zero-value wormhole exits bypass every fee floor, letting a free settlement consume 83% of a block and permanently occupy 7 nullifier slots per spend
* \#89950 \[BC-High] process\_exit\_bundle never calls record\_transfer for miner\_fee: every wormhole miner-fee credit is permanently unspendable
* \#89306 \[BC-High] Reward leaves commit `floor(amount / 1e10)` with no remainder carry
* \#89613 \[BC-High] QPoW import-queue verifier does not check work, so an anonymous peer holds an authority in major sync and halts its block production indefinitely at zero hash power
* \#89139 \[BC-High] Non-quantized credits to wormhole addresses permanently lock sub-quantum value
* \#89689 \[BC-High] Miner Volume-Fee and Aggregator Rebate Credits Lack ZK-Tree Leaves, Permanently Freezing Funds

</details>

<details>

<summary>Medium</summary>

* \#88855 \[BC-Medium] `base_extrinsic` is Substrate's stock sr25519 constant, signed extrinsic is undercharged
* \#90466 \[BC-Medium] A failed enactment scheduling attempt destroys pallet-referenda's preimage reference, so an approved referendum's Root call is silently lost
* \#89524 \[BC-Medium] retarget\_schedule silently reassigns a beneficiary's already-vested funds to their replacement
* \#89492 \[BC-Medium] Mismatched dest can\_deposit check in fungible transfer leads to accepted transfers that credit nothing
* \#90210 \[BC-Medium] Quantus's 36,772-transaction default crosses the 10,239-entry gossip cache, causing repeated full-peer disconnects and node resource amplification
* \#88939 \[BC-Medium] Unauthenticated PQ Handshakes Increase Validator CPU Consumption by \~39% Before Peer Authentication and Established-Connection Accounting
* \#89213 \[BC-Medium] Permanent guardian enrollment griefing via missing OnKilledAccount cleanup
* \#88507 \[BC-Medium] Missing guardian consent in set\_high\_security lets anyone permanently exhaust another account's guardian slots
* \#90408 \[BC-Medium] Quantus's safe RPC exposes GenesisBuilder with unbounded derivation paths, letting unauthenticated callers pin node CPU cores
* \#89478 \[W\&A-Medium] Balance-privacy display blanks the transfer amount in the multisig approval confirmation, enabling blind co-signer approval and direct theft of user funds
* \#88218 \[BC-Medium] Per-extrinsic base weight is understated by 12.39×: the runtime inherits upstream Substrate's sr25519 constant while verifying post-quantum Dilithium signatures
* \#89307 \[BC-Medium] Sub-existential-deposit block fees are minted before the account-endowing reward
* \#88736 \[BC-Medium] Missing pending-handshake admission cap in the Quantus PQ Noise stack lets unauthenticated peers force ML-KEM/ML-DSA work and 65 KB allocations per connection, increasing node res...
* \#89154 \[BC-Medium] Sparse public batches retain 742-exit weight for one exit, letting an unsigned proof stuff 83% of the Normal quota
* \#90360 \[BC-Medium] Unauthenticated 28-byte /noise negotiation forces ML-KEM and ML-DSA work before peer authentication
* \#89284 \[BC-Medium] Permissionless high-security enrollment lets attackers irreversibly exhaust an arbitrary guardian's account capacity
* \#88215 \[BC-Medium] Any account can permanently and irrevocably consume another address's bounded guardian index, with no consent and no removal path
* \#88219 \[BC-Medium] Blocks with invalid proof-of-work carry no peer penalty: a hostile peer can force unlimited sync restarts and is re-admitted as a stranger every time
* \#88221 \[W\&A-Medium] Unvalidated `amount` in the /pay deep link freezes and crashes the wallet via unbounded BigInt materialisation on the UI isolate
* \#88224 \[BC-Medium] A peer advertising a forged block height suspends a victim node's mining AND silently blackholes its block-import notifications
* \#88222 \[BC-Medium] One signer can permanently saturate any account's guardian-enrollment capacity, without consent, using unfunded derivative origins
* \#88227 \[BC-Medium] Incomplete degenerate-key filter in ML-DSA verify\_var allows keyless signature forgery for a 63-value family of t1 coefficients
* \#88226 \[BC-Medium] Two unauthenticated bytes make a node commit 16 KiB of resident memory, in the pre-handshake multistream-select framing path
* \#88230 \[BC-Medium] PoW seal verified in import\_block instead of the Verifier: bad-seal blocks from a peer trigger an unpenalised sync-restart loop, stalling block import and mining
* \#90220 \[BC-Medium] Dilithium signature verification is undercharged by stale base extrinsic weight
* \#88249 \[BC-Medium] ExtrinsicBaseWeight is still upstream's sr25519 measurement after the switch to ML-DSA, so every signed extrinsic is charged 12% of what it costs
* \#88251 \[BC-Medium] The per-extrinsic weight ledger undercharges by 91.9%: the runtime inherits upstream Substrate's sr25519 constant while verifying post-quantum Dilithium signatures in WASM
* \#88274 \[W\&A-Medium] With Hide balance enabled, a Utility.batch multisig proposal shows no transfer amount anywhere on the approval sheet, while the same transfer unwrapped still shows it
* \#89322 \[W\&A-Medium] Call parsing in quantus\_sdk freezes mobile wallet
* \#89889 \[BC-Medium] Permissionless vesting claim strands a schedule in an un-terminable state, blocking the treasury's clawback of the final tranche
* \#88768 \[BC-Medium] Zero-cost mempool flooding via fee-free unsigned wormhole-exit admission causes network processing nodes to process transactions beyond set parameters
* \#89353 \[BC-Medium] Unbounded pre-auth PQ Noise setup saturates full-node CPU before established-peer limits apply
* \#88529 \[BC-Medium] Incorrect base-extrinsic weight for ML-DSA-87 undercharges transaction verification by \~70%
* \#89137 \[BC-Medium] Anyone can permanently exhaust any account's guardian slots for \~0.032 UNIT, blocking it from ever protecting another account
* \#89762 \[BC-Medium] Owner-cancelling a reversible transfer mints an unbacked ZK-tree leaf (permanent, permissionless state growth)
* \#88329 \[BC-Medium] Missing guardian consent in set\_high\_security lets one signer permanently exhaust any guardian's quota and block legitimate enrollments
* \#89091 \[BC-Medium] Quantus adopts an sr25519-calibrated base weight for ML-DSA signatures, undercharging signed extrinsics by 79.58%
* \#88786 \[W\&A-Medium] Official Quantus POS marks an attacker-crafted, rolled-back batch\_all payment as received, in violation of the node's documented integration requirement
* \#88633 \[BC-Medium] ExtrinsicBaseWeight does not account for ML-DSA-87 verification — every signed extrinsic undercharges node CPU by at least 55.7%
* \#88386 \[BC-Medium] Unsubscribe lifecycle bug bypasses txWatch RPC caps and increases full-node resource use by 121%
* \#88397 \[BC-Medium] Self-directed reversible transfers record wormhole ZK-tree leaves for credits that never happen
* \#88671 \[BC-Medium] Permissionless vesting claim forces payout fragmentation, leading to loss of beneficiary funds
* \#89188 \[W\&A-Medium] Official UTXO pager orders only by block height, skips a same-height sibling, and claim never sees that note
* \#90047 \[BC-Medium] Unbounded SURI path depth bypasses MAX\_DEV\_ACCOUNTS via public /light/2 GenesisBuilder\_build\_state, consuming \~190 CPU-seconds per 34 KB unauthenticated P2P request
* \#90343 \[BC-Medium] Txpool skips maintain on every finality event: watched txs never see Finalized and are falsely timed out after 512 blocks despite being finalized

</details>

<details>

<summary>Low</summary>

* \#90448 \[BC-Low] recover\_funds is priced by transfer count while its real cost scales with scheduler-agenda density
* \#90496 \[BC-Low] One 1,000-QUAN balance advertises 1,838,500 QUAN of unreserved tips to monopolize each targeted node's future queue
* \#89475 \[BC-Low] Missing zk-tree leaf record in the miner volume-fee mint leads to permanent freezing of the miner's fee share
* \#89809 \[W\&A-Low] Nested `Utility.if_else` proposal forces exponential work in the official SDK and temporarily disables a victim's multisig view
* \#89219 \[BC-Low] Missing Merkle-Leaf Record Lead to Permanent Freezing of the Miner Volume Fee
* \#89310 \[BC-Low] Author-steered timestamp delta biases the difficulty retarget downward
* \#90377 \[BC-Low] Stale transaction-payment benchmark omits Quantus fee-collector storage, undercharging the extension by 48.12% in the measured run
* \#88568 \[BC-Low] Missing collector storage weight in TransactionFeesCollector leads to a whole-extrinsic undercharge of at least 16.7160%
* \#88263 \[W\&A-Low] A late check-phrase lookup overwrites the current recipient's phrase, so the send review screen vouches for an address the user is not paying
* \#89077 \[BC-Low] Incorrect genesis balance accounting in Wormhole on\_initialize allows reserved/locked funds to be exited as free balance, leading to direct loss of funds and unauthorized minting
* \#88305 \[W\&A-Low] Quantus SDK predicts a multisig address the chain never creates, because its derivation omits the pallet's signer deduplication
* \#89098 \[BC-Low] Multisig execute wraps transfers at zero proof-recording weight
* \#88830 \[BC-Low] Missing constraint. private-batch allows real outputs with canonical zero exit account
* \#89848 \[BC-Low] Scheduled reversible transfer can become stuck in a held state after transfer\_keep\_alive failure (pallet-reversible-transfers)
* \#90369 \[BC-Low] A failed enactment placement destroys an approved referendum's proposal, so the voted Root call never executes

</details>

<details>

<summary>Insight</summary>

* \#88382 \[W\&A-Insight] Wallet mnemonic stored with default iOS Keychain accessibility (kSecAttrAccessibleWhenUnlocked) — seed is backup-extractable and not device-bound
* \#88243 \[BC-Insight] check\_wormhole\_path accepts any path shape and hdwallet exports no path constructor, so the node CLI and the mobile SDK derive different wormhole keys for the same index
* \#88290 \[BC-Insight] Non-canonical protobuf-based PeerId derivation in litep2p's Noise handshake leads to reputation-ban evasion in Peerstore

</details>

## Reports by Type

<details>

<summary>Blockchain/DLT</summary>

* \#90448 \[BC-Low] recover\_funds is priced by transfer count while its real cost scales with scheduler-agenda density
* \#88855 \[BC-Medium] `base_extrinsic` is Substrate's stock sr25519 constant, signed extrinsic is undercharged
* \#90466 \[BC-Medium] A failed enactment scheduling attempt destroys pallet-referenda's preimage reference, so an approved referendum's Root call is silently lost
* \#89058 \[BC-High] pallet-multisig decodes an opaque inner call with no depth limit: a 4.8 KB nested blob traps the runtime before the weight guard can fire, so the work is never charged
* \#90496 \[BC-Low] One 1,000-QUAN balance advertises 1,838,500 QUAN of unreserved tips to monopolize each targeted node's future queue
* \#89475 \[BC-Low] Missing zk-tree leaf record in the miner volume-fee mint leads to permanent freezing of the miner's fee share
* \#89733 \[BC-High] Wormhole miner volume fee is credited to a keyless account without a ZK-tree leaf, permanently freezing it
* \#89524 \[BC-Medium] retarget\_schedule silently reassigns a beneficiary's already-vested funds to their replacement
* \#89552 \[BC-High] QPoW miners will permanently freeze the sub-quantum remainder of every block reward because `mint_reward()` credits full planck while `hash_leaf()` only commits \`floor(amount / 10^1...
* \#89492 \[BC-Medium] Mismatched dest can\_deposit check in fungible transfer leads to accepted transfers that credit nothing
* \#90398 \[BC-High] An unauthenticated attacker will force processing nodes to process unsigned Wormhole mempool transactions beyond configured proposer limits.
* \#90400 \[BC-High] Opaque multisig proposals bypass the 256-level extrinsic depth limit and repeatedly trap block-author Wasm without a durable fee or nonce charge
* \#89040 \[BC-High] Wormhole miner-fee credit records no zk-tree leaf, freezing miner yield
* \#89197 \[BC-High] Missing ZK-leaf recording permanently freezes miner-fee balances credited to keyless Wormhole accounts
* \#88485 \[BC-High] Unverified nullifier public inputs bypass Wormhole semantic txpool deduplication
* \#90210 \[BC-Medium] Quantus's 36,772-transaction default crosses the 10,239-entry gossip cache, causing repeated full-peer disconnects and node resource amplification
* \#89984 \[BC-High] Missing ZK-tree registration for miner volume fees permanently freezes credited funds
* \#88939 \[BC-Medium] Unauthenticated PQ Handshakes Increase Validator CPU Consumption by \~39% Before Peer Authentication and Established-Connection Accounting
* \#89213 \[BC-Medium] Permanent guardian enrollment griefing via missing OnKilledAccount cleanup
* \#88507 \[BC-Medium] Missing guardian consent in set\_high\_security lets anyone permanently exhaust another account's guardian slots
* \#88943 \[BC-High] Residue truncation and two-leaf split in pallet-mining-rewards lead to permanent freezing of unclaimed miner yield
* \#88951 \[BC-High] Batch-level fee rounding permanently strands fees locked by Wormhole leaf proofs
* \#89053 \[BC-High] Wormhole miner volume fees are permanently unclaimable because the credit creates no ZK-tree leaf
* \#89250 \[BC-High] Missing quantum-alignment in the wormhole deposit chokepoint leads to permanent freezing of funds
* \#88938 \[BC-High] Missing ZK-tree leaf record in the wormhole miner volume-fee credit path leads to permanent freezing of funds
* \#90207 \[BC-Critical] Missing ZK-tree leaf on the wormhole miner-fee credit leads to permanent freezing of the block author's volume fee
* \#90408 \[BC-Medium] Quantus's safe RPC exposes GenesisBuilder with unbounded derivation paths, letting unauthenticated callers pin node CPU cores
* \#90180 \[BC-High] Wormhole miner volume-fee credits are uncollectible because no proof leaf is recorded
* \#88609 \[BC-Critical] High-security whitelist is incomplete: the transaction tip bypasses the guardian delay entirely
* \#88218 \[BC-Medium] Per-extrinsic base weight is understated by 12.39×: the runtime inherits upstream Substrate's sr25519 constant while verifying post-quantum Dilithium signatures
* \#89307 \[BC-Medium] Sub-existential-deposit block fees are minted before the account-endowing reward
* \#88571 \[BC-High] Missing ZK-tree leaf on the wormhole exit miner-fee credit permanently freezes all miner exit-fee revenue (keyless account, no exit path)
* \#88736 \[BC-Medium] Missing pending-handshake admission cap in the Quantus PQ Noise stack lets unauthenticated peers force ML-KEM/ML-DSA work and 65 KB allocations per connection, increasing node res...
* \#88735 \[BC-High] Lossy Wormhole leaf quantization permanently freezes a portion of every non-aligned mining reward and transaction-fee payout
* \#89154 \[BC-Medium] Sparse public batches retain 742-exit weight for one exit, letting an unsigned proof stuff 83% of the Normal quota
* \#88300 \[BC-High] Missing Wormhole ZK leaf permanently freezes all miner exit-fee yield
* \#89304 \[BC-High] Wormhole miner volume fee is credited without a ZK leaf and can never be spent
* \#89219 \[BC-Low] Missing Merkle-Leaf Record Lead to Permanent Freezing of the Miner Volume Fee
* \#89310 \[BC-Low] Author-steered timestamp delta biases the difficulty retarget downward
* \#90360 \[BC-Medium] Unauthenticated 28-byte /noise negotiation forces ML-KEM and ML-DSA work before peer authentication
* \#90377 \[BC-Low] Stale transaction-payment benchmark omits Quantus fee-collector storage, undercharging the extension by 48.12% in the measured run
* \#88540 \[BC-High] Wormhole miner volume fees are permanently unclaimable because no ZK leaf is created
* \#89284 \[BC-Medium] Permissionless high-security enrollment lets attackers irreversibly exhaust an arbitrary guardian's account capacity
* \#88215 \[BC-Medium] Any account can permanently and irrevocably consume another address's bounded guardian index, with no consent and no removal path
* \#88219 \[BC-Medium] Blocks with invalid proof-of-work carry no peer penalty: a hostile peer can force unlimited sync restarts and is re-admitted as a stranger every time
* \#88224 \[BC-Medium] A peer advertising a forged block height suspends a victim node's mining AND silently blackholes its block-import notifications
* \#88222 \[BC-Medium] One signer can permanently saturate any account's guardian-enrollment capacity, without consent, using unfunded derivative origins
* \#88227 \[BC-Medium] Incomplete degenerate-key filter in ML-DSA verify\_var allows keyless signature forgery for a 63-value family of t1 coefficients
* \#88226 \[BC-Medium] Two unauthenticated bytes make a node commit 16 KiB of resident memory, in the pre-handshake multistream-select framing path
* \#88230 \[BC-Medium] PoW seal verified in import\_block instead of the Verifier: bad-seal blocks from a peer trigger an unpenalised sync-restart loop, stalling block import and mining
* \#90235 \[BC-High] Wormhole fee credits skip record\_transfer: miner and aggregator volume fees are minted to keyless addresses with no zk-tree leaf, permanently freezing them
* \#88243 \[BC-Insight] check\_wormhole\_path accepts any path shape and hdwallet exports no path constructor, so the node CLI and the mobile SDK derive different wormhole keys for the same index
* \#90220 \[BC-Medium] Dilithium signature verification is undercharged by stale base extrinsic weight
* \#88249 \[BC-Medium] ExtrinsicBaseWeight is still upstream's sr25519 measurement after the switch to ML-DSA, so every signed extrinsic is charged 12% of what it costs
* \#88250 \[BC-High] One unauthenticated TCP burst permanently terminates a node's entire networking stack, with no recovery short of a process restart
* \#88251 \[BC-Medium] The per-extrinsic weight ledger undercharges by 91.9%: the runtime inherits upstream Substrate's sr25519 constant while verifying post-quantum Dilithium signatures in WASM
* \#88566 \[BC-High] PR #656 incomplete remediation: bundle-level rounding under-settles per-leaf Wormhole fees
* \#89065 \[BC-High] Sub-fee-exitable wormhole deposits are accepted and recorded but can never be spent or exited, permanently freezing user funds
* \#88568 \[BC-Low] Missing collector storage weight in TransactionFeesCollector leads to a whole-extrinsic undercharge of at least 16.7160%
* \#89889 \[BC-Medium] Permissionless vesting claim strands a schedule in an un-terminable state, blocking the treasury's clawback of the final tranche
* \#90329 \[BC-High] pallet\_multisig::propose decodes opaque inner calls without a depth limit, trapping the block producer
* \#88768 \[BC-Medium] Zero-cost mempool flooding via fee-free unsigned wormhole-exit admission causes network processing nodes to process transactions beyond set parameters
* \#88290 \[BC-Insight] Non-canonical protobuf-based PeerId derivation in litep2p's Noise handshake leads to reputation-ban evasion in Peerstore
* \#89077 \[BC-Low] Incorrect genesis balance accounting in Wormhole on\_initialize allows reserved/locked funds to be exited as free balance, leading to direct loss of funds and unauthorized minting
* \#88771 \[BC-High] Wormhole miner volume-fee is credited to a key-less address with no ZK-tree leaf, permanently freezing it
* \#89134 \[BC-High] Miner volume-fee credits lack ZK-tree leaves, permanently freezing block-author fees
* \#89353 \[BC-Medium] Unbounded pre-auth PQ Noise setup saturates full-node CPU before established-peer limits apply
* \#89171 \[BC-High] Missing pending-connection cap and per-IP limit in litep2p TCP transport causes 30%+ resource increase and 500%+ block delays via zombie peer flood
* \#88529 \[BC-Medium] Incorrect base-extrinsic weight for ML-DSA-87 undercharges transaction verification by \~70%
* \#88297 \[BC-High] Scheduled transfers below the existential deposit strand the sender's funds with no owner-accessible recovery
* \#89137 \[BC-Medium] Anyone can permanently exhaust any account's guardian slots for \~0.032 UNIT, blocking it from ever protecting another account
* \#89347 \[BC-Critical] High-security accounts can be drained immediately via an unbounded tip on a whitelisted call
* \#89348 \[BC-High] Mining reward's zk-tree leaf floors the amount, freezing the remainder
* \#89762 \[BC-Medium] Owner-cancelling a reversible transfer mints an unbacked ZK-tree leaf (permanent, permissionless state growth)
* \#88329 \[BC-Medium] Missing guardian consent in set\_high\_security lets one signer permanently exhaust any guardian's quota and block legitimate enrollments
* \#88969 \[BC-High] Unbounded recursive decoding of opaque multisig calls makes block authors repeatedly execute unpaid mempool poison
* \#89091 \[BC-Medium] Quantus adopts an sr25519-calibrated base weight for ML-DSA signatures, undercharging signed extrinsics by 79.58%
* \#89098 \[BC-Low] Multisig execute wraps transfers at zero proof-recording weight
* \#88628 \[BC-Critical] `Balances::transfer_allow_death` debits the sender, credits the recipient nothing, and reports success
* \#88498 \[BC-High] Non-quantized mining payouts permanently strand approximately 0.786% of block-author yield
* \#89062 \[BC-High] Remote unauthenticated block-production DoS: fee-free unverifiable wormhole exit proofs force unbounded failed ZK verifications on every block producer (up to the full 9s build budg...
* \#88633 \[BC-Medium] ExtrinsicBaseWeight does not account for ML-DSA-87 verification — every signed extrinsic undercharges node CPU by at least 55.7%
* \#90133 \[BC-High] Mining rewards are minted at raw precision but their ZK leaves commit only `floor(amount / 0.01 QUAN)`, permanently freezing every remainder and making ordinary fee yield wholly unc...
* \#89438 \[BC-Critical] Miner-controlled future timestamps bypass reversible-transfer cancellation windows and cause direct loss of funds
* \#88830 \[BC-Low] Missing constraint. private-batch allows real outputs with canonical zero exit account
* \#89448 \[BC-High] Block authors are paid wormhole volume fees at a keyless address that no exit proof can unlock
* \#89449 \[BC-High] Mining rewards and fee credits mint more than their ZK leaf commits, stranding the difference at a keyless miner address
* \#88841 \[BC-Critical] High-security accounts are drained irreversibly through the transaction tip, which no high-security check in the runtime models: the guardian delay, cancel and recover\_funds are...
* \#88386 \[BC-Medium] Unsubscribe lifecycle bug bypasses txWatch RPC caps and increases full-node resource use by 121%
* \#88397 \[BC-Medium] Self-directed reversible transfers record wormhole ZK-tree leaves for credits that never happen
* \#88404 \[BC-High] Preimage reference over-drop in referenda enactment scheduling silently voids a passed governance decision
* \#88671 \[BC-Medium] Permissionless vesting claim forces payout fragmentation, leading to loss of beneficiary funds
* \#88872 \[BC-High] Missing `record_transfer` on the miner volume-fee credit permanently freezes unclaimed miner yield on a non-ML-DSA-signable QPoW wormhole author
* \#88881 \[BC-High] Wormhole miner volume fee is minted without its ZK-tree leaf, permanently freezing every exit fee.
* \#88223 \[BC-High] Miner volume fee and aggregator rebate are credited to keyless wormhole addresses with no zk-tree leaf, permanently freezing them (incomplete fix of EQ-QNT-WORMHOLE-F-03)
* \#89009 \[BC-High] Miner's share of the wormhole exit volume fee is credited to a keyless address with no ZK leaf, permanently freezing it
* \#89163 \[BC-High] Miner volume fee is reported as paid but records no ZK leaf, so the credited fee is unrealizable by the miner
* \#89036 \[BC-High] Unbounded unauthenticated PQ handshakes permanently terminate node P2P networking and can halt network-wide transaction confirmation
* \#88912 \[BC-High] Missing `record_transfer` in Wormhole miner-fee settlement permanently freezes unclaimed yield
* \#89247 \[BC-High] Zero-value wormhole exits bypass every fee floor, letting a free settlement consume 83% of a block and permanently occupy 7 nullifier slots per spend
* \#89819 \[BC-Critical] Missing FIPS 204 domain separation in ML-DSA transaction verification lets a victim's message signature authorize a transfer, enabling direct theft of user funds
* \#90047 \[BC-Medium] Unbounded SURI path depth bypasses MAX\_DEV\_ACCOUNTS via public /light/2 GenesisBuilder\_build\_state, consuming \~190 CPU-seconds per 34 KB unauthenticated P2P request
* \#89950 \[BC-High] process\_exit\_bundle never calls record\_transfer for miner\_fee: every wormhole miner-fee credit is permanently unspendable
* \#89306 \[BC-High] Reward leaves commit `floor(amount / 1e10)` with no remainder carry
* \#89832 \[BC-Critical] Unbounded transaction tips bypass high-security containment and drain compromised accounts
* \#89613 \[BC-High] QPoW import-queue verifier does not check work, so an anonymous peer holds an authority in major sync and halts its block production indefinitely at zero hash power
* \#89139 \[BC-High] Non-quantized credits to wormhole addresses permanently lock sub-quantum value
* \#89848 \[BC-Low] Scheduled reversible transfer can become stuck in a held state after transfer\_keep\_alive failure (pallet-reversible-transfers)
* \#89689 \[BC-High] Miner Volume-Fee and Aggregator Rebate Credits Lack ZK-Tree Leaves, Permanently Freezing Funds
* \#89136 \[BC-Critical] Missing deposit-release calls in the high-security whitelist permanently freeze reserved balances, including an uninvolved third party's recovery deposit
* \#90343 \[BC-Medium] Txpool skips maintain on every finality event: watched txs never see Finalized and are falsely timed out after 512 blocks despite being finalized
* \#90369 \[BC-Low] A failed enactment placement destroys an approved referendum's proposal, so the voted Root call never executes

</details>

<details>

<summary>Websites &#x26; Applications</summary>

* \#90460 \[W\&A-High] Quantus POS accepts a replaceable pool transaction as payment before execution
* \#89657 \[W\&A-High] Validation gap in multisig confirmation allows approval of hidden transfer arguments
* \#89806 \[W\&A-High] Frozen Vesting decode failure leaves an opaque multisig approval actionable and allows a 999.999 QUAN drain
* \#89808 \[W\&A-High] Unit `AccountIndex` consumes zero runtime bytes while the SDK consumes the next field, allowing one canonical approval to display no attacker transfer while transferring 999.999 QU...
* \#90351 \[W\&A-High] The multisig approval screen signs call bytes it could not decode, showing the signer a call name and no parameters
* \#89987 \[W\&A-High] MultiAddress::Index is decoded at a different width by the SDK and the runtime, letting a QR code display a harmless batch while signing a 1000 QUAN transfer to the attacker
* \#89283 \[W\&A-High] Runtime-metadata interpretation conflict in Quantus SDK lets a multisig signer conceal `transfer_all` and drain shared funds
* \#89809 \[W\&A-Low] Nested `Utility.if_else` proposal forces exponential work in the official SDK and temporarily disables a victim's multisig view
* \#88914 \[W\&A-High] Hide Balances masks executable multisig amounts, enabling deceptive `utility.if_else` approvals
* \#89478 \[W\&A-Medium] Balance-privacy display blanks the transfer amount in the multisig approval confirmation, enabling blind co-signer approval and direct theft of user funds
* \#89798 \[W\&A-High] Multisig review hides call parameters and enables a treasury drain
* \#88382 \[W\&A-Insight] Wallet mnemonic stored with default iOS Keychain accessibility (kSecAttrAccessibleWhenUnlocked) — seed is backup-extractable and not device-bound
* \#88221 \[W\&A-Medium] Unvalidated `amount` in the /pay deep link freezes and crashes the wallet via unbounded BigInt materialisation on the UI isolate
* \#88263 \[W\&A-Low] A late check-phrase lookup overwrites the current recipient's phrase, so the send review screen vouches for an address the user is not paying
* \#88274 \[W\&A-Medium] With Hide balance enabled, a Utility.batch multisig proposal shows no transfer amount anywhere on the approval sheet, while the same transfer unwrapped still shows it
* \#89322 \[W\&A-Medium] Call parsing in quantus\_sdk freezes mobile wallet
* \#88305 \[W\&A-Low] Quantus SDK predicts a multisig address the chain never creates, because its derivation omits the pallet's signer deduplication
* \#89999 \[W\&A-Critical] Multisig discovery accepts an indexer-supplied address without verifying its derivation
* \#89095 \[W\&A-High] Android process death defeats recovery-phrase clipboard expiry and exposes the mnemonic to later foreground apps
* \#88786 \[W\&A-Medium] Official Quantus POS marks an attacker-crafted, rolled-back batch\_all payment as received, in violation of the node's documented integration requirement
* \#88348 \[W\&A-High] Exponential synchronous decoding of a 213-byte Utility.if\_else proposal leaves the mobile wallet unable to render a frame or accept input whenever the affected multisig account loads
* \#88483 \[W\&A-High] Official Multisig.approve displays a zero-value call and still signs a full vault transfer
* \#88851 \[W\&A-High] Exponential `Utility::if_else` call display lets a multisig co-signer persistently freeze the mobile wallet
* \#88667 \[W\&A-High] Unhandled call-decode failure in the multisig approval sheet leads to direct theft of user funds
* \#89151 \[W\&A-High] Primary-only Utility.if\_else approval summary shows 1 QUAN while a 50 QUAN fallback drains the multisig.
* \#89008 \[W\&A-High] Fail-open multisig call decoding in the mobile app hides transfer\_all and enables theft of the full multisig balance
* \#89011 \[W\&A-High] Fail-open runtime-call decoding lets a minority multisig signer obtain approval for full-balance theft
* \#89188 \[W\&A-Medium] Official UTXO pager orders only by block height, skips a same-height sibling, and claim never sees that note
* \#88730 \[W\&A-Critical] Generated `MultiAddress::Index` decodes Unit as Compact, letting one signer hide `transfer_all` in an authoritative 2-of-2 multisig proposal and drain the multisig
* \#89007 \[W\&A-Critical] Accessibility bypass of the wallet lock screen permits an unauthenticated sweep of the entire wormhole balance
* \#89802 \[W\&A-Critical] Missing chain-identity validation in the Quantus SDK signing path lets a malicious RPC endpoint obtain signatures valid on an unintended chain

</details>
