# 89062 bc high remote unauthenticated block production dos fee free unverifiable wormhole exit proofs force unbounded failed zk verifications on every block producer up to the full 9s build budg&#x20;

**Submitted on Aug 16th 2026 at 22:47:20 UTC by @craftsoldier for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89062
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

**Vulnerability.** The wormhole system lets users withdraw from private balances by submitting a ZK proof as an unsigned, fee-free transaction, allowing anyone to submit a proof without an account, stake or signature. The intended flow is to let the network check the proof at block production and mint the funds only if it is valid.

Currently the checking is split in two; when a transaction first arrives, pool admission runs only cheap structural checks (about 0.9 ms) on the length, deserialization, encoding format, and that the referenced block exists. The expensive ZK verification, about 15.16 ms in wasm (`PRIVATE_BATCH_ZK_VERIFY_REF_TIME_PS` in `pallets/wormhole/src/weights.rs`), only runs later, when a block producer attempts to include the transaction in a block.

**The bug lies in the missing limit on how many failed proof verifications a block producer can be forced to perform per block.** When verification fails, the producer discards the transaction and moves straight to the next one. A rejection here consumes no block weight and does not count toward the loop's stopping mechanism, `MAX_SKIPPED_TRANSACTIONS` in `sc-basic-authorship-0.53.0/src/basic_authorship.rs`, because that counter only tracks transactions that exhaust block resources, not ones that fail checks. The producer therefore keeps verifying candidates one by one until it hits its 9 second build deadline, which is 90 percent of the 10 second build window set in `node/src/service.rs`. Anyone who can fill the pool with candidates that pass the cheap checks but fail the expensive one makes every producer spend its entire build budget, on every block, on verification work that yields nothing.

**Attack.** Any network peer can take the committed real private batch proof (`pallets/wormhole/test-data/private_batch.hex`, 157500 bytes) and rewrite only its public inputs. Two small edits are needed: the header must point at a recent block, any of the last 4096 works, and the 28 nullifier field elements must be randomized. Admission to the pool is free, so every peer on the network can produce these at well - no account, balance or signature is ever needed allowing any peer to generate unlimited copies.

The genuine proof body passes every admission check, and the random nullifiers make each copy look like a brand new exit nobody has spent, which dodges both pool dedup and the freshness check. At the inclusion gate the **entire ZK verification runs**, all internal gates pass, and only the final public input check fails, so every rejection costs the producer the full verify. Each junk candidate is admission cheap and inclusion expensive, which is exactly the asymmetry being exploited.

The cost of sustaining the attack is dangerously negligible relative to the damage inflicted. The default pool limits of 256 MiB and 36772 transactions, set in `client/cli/src/params/transaction_pool_params.rs` lines 48 and 54, hold about 1700 such proofs. About 593 proofs or 93 MB saturate a full 9 second window on reference hardware, and sustaining costs about 8 MB per second. Both entry surfaces were verified, libp2p gossip from a peer node and the `author_submitExtrinsic` RPC on the producer itself.

**Impact.** While the pool is saturated, every producer spends its entire build budget on verifications that all fail, and on the reference hardware the chain's own weights are priced against, that is the full 9 second budget on every single block. This lasts exactly as long as the attacker keeps up the stream, one VPS and about 8 MB per second, since no on-chain mechanism can make it expensive or traceable. Within the window the burn is worse than once per block, because block production also rebuilds on every transaction arrival, rate limited to one per 2 seconds, and each rebuild walks the queue again. The moment the stream stops, the pool drains on its own and cadence recovers within about a minute, which the PoC measured directly.

Blocks continue to be produced, so this is not a halt and no funds are at risk. But block times stretch from a 12 second target to over a minute, confirming transactions wait 4x longer, and **most user traffic cannot enter the mempool at all** because it is full of junk. Every producer on the network eats this simultaneously, from a single anonymous peer.

Live verified on a local two node dev cluster against stock HEAD `94c4ca7a`: the producer ran **45744 full ZK verifications in 3 minutes**, all failed and all discarded, zero junk entered any block, block builds went from about 0 to 2 ms to a **median of 2837 ms and a max of 7668 ms**, which is 85 percent of the 9 second budget, block intervals stretched to a **mean of 43.4 s and a max of 142.4 s** against a 12 s target, and signed transactions still confirmed but at **81.3 s mean latency** against 21.8 s at rest.

Severity mapping: temporary freezing of network transactions by a 500 percent or more block time delay, processing mempool transactions beyond set parameters, and a resource consumption increase above 30 percent, which is **High**.

## Link to Proof of Concept

<https://gist.github.com/craftsoldier/18a86f39a364aa239d8968f4c3754f14>

## Proof of Concept

All testing was done on a local two node dev cluster.

{% stepper %}
{% step %}

### Build and start the nodes

Build the vulnerable node from stock HEAD `94c4ca7a`, no modifications:

```bash
cargo build --release -p quantus-node
```

Start the producer:

```bash
quantus-node --dev
```

Built-in CPU miner, p2p `:30333`, RPC `:9944`.

Start the attacker:

```bash
quantus-node --chain dev --bootnodes <producer>
```

P2P `:30334`, RPC `:9945`, a plain full node, not a validator, no keys. It just peers with the producer over libp2p. That peer is the entire attack position.
{% endstep %}

{% step %}

### Establish a baseline

Let the chain idle for 45 seconds. Every block build takes 0 to 2 ms, visible in the producer log as `Prepared block for proposing at N (0 ms)`.

Send two ordinary signed transactions with a 1 UNIT tip; they confirm in about 22 seconds at dev difficulty. These are the control numbers. On this chain a signed transaction is a Dilithium87 signed extrinsic carrying the standard 13 extension tuple, and any dev genesis funded account works as sender, the easiest source is a small script against the runtime crates, but any existing wallet does.
{% endstep %}

{% step %}

### Construct the junk without cryptography

Take the real private batch proof that ships in the repo (`pallets/wormhole/test-data/private_batch.hex`, 157500 bytes) and change only the trailing 1248 bytes of public inputs; the 156252 byte proof body stays untouched.

The public input section is `u64 count (8 bytes) then 155 field elements (8 bytes each, little endian)`. Its layout is: felt 0 is the exit slot count, felt 1 the asset id, felt 2 the fee rate, felts 3 to 6 the block hash as four limbs, felt 7 the block number, felts 8 to 77 the exit slots as 14 groups of `(amount, account as 4 limbs)`, felts 78 to 105 the nullifiers as 7 groups of 4 limbs, felts 106 to 154 zero padding.

Three edits, everything else copied verbatim: set felts 3 to 7 to a recent block, any of the last 4096 works, set the 28 nullifier limbs to fresh random values, leave the exit slots alone. Two constraints: every value must stay below the Goldilocks modulus `0xFFFFFFFF00000001` (the four block hash limbs must be checked against it, pick a different recent block if one limb is not canonical), and the total length must stay exactly 157500 bytes so the canonical round trip still matches.

```
junk = real_proof_body || [pi_count=155] || [header(recent_block)] ||
       [real exit slots] || [random nullifiers x 28] || [zero padding]
xt   = bare unsigned extrinsic wrapping Wormhole::verify_private_batch{ proof_bytes: junk }
```

The bare unsigned extrinsic is just the SCALE encoding, a length prefixed byte 4 (bare, version 4) followed by the call bytes, which are the pallet and function indices (wormhole, `verify_private_batch`) followed by the length prefixed proof bytes. Submit about 1400 of these, half through the attacker node RPC so they travel to the producer as gossip, half straight into the producer `author_submitExtrinsic` RPC, then keep refilling at roughly 300 per second for 3 minutes. The refill matters because every rejected candidate gets banned and evicted, the pool drains on its own unless fed.
{% endstep %}

{% step %}

### Observe producer behavior

On arrival, admission runs only the cheap checks: length, deserialize, canonical round trip, public input parse, and bundle checks. All pass, the proof body is genuine and the public inputs are structurally valid. The random nullifiers give every copy a distinct `provides` tag, so the pool never dedups. The candidate sits in the ready queue looking legitimate.

The block producer walks the queue and each candidate hits `pre_dispatch`, which runs the **full ZK verification**. Every internal gate passes, the body is real, and only the final public input check fails. The producer logs `Private-batch proof verification failed` and moves on to the next candidate.

That “moves on” is the whole problem. The rejection is `InvalidTransaction::Call`, not `exhausted_resources`, so it never increments the `skipped` counter that would normally stop the walk after 8 bad candidates, and it consumes no block weight because failed unsigned extrinsics pay nothing. The loop keeps verifying, one 15 ms corpse at a time, until the queue empties or the 9 second hard deadline hits. This repeats on every block, plus on every rebuild, which triggers per transaction arrival at up to one per 2 seconds.
{% endstep %}
{% endstepper %}

## Observed result: 3 minute attack

* **45744 failed full verifications**, one per junk candidate, and **zero junk extrinsics in any block** (every block body was checked). All of that compute bought nothing.
* Block builds went from 0 to 2 ms to a **median of 2837 ms and a max of 7668 ms**, 85 percent of the 9 second budget.
* Block intervals stretched from about 7 seconds to a **mean of 43.4 s and a max of 142.4 s** against a 12 second target. Difficulty never moved, so this is not a difficulty effect.
* The signed canaries still confirmed, priority ordering puts them above the unsigned junk, but at **81.3 s mean latency** instead of 21.8 s.
* Producer CPU pinned near 99 percent the whole time, attacker node idle. Nothing crashed, blocks kept coming, just slow and empty.

## Hardware caveat

A full verify on this Apple Silicon machine takes about 2.2 ms in compiled wasm, not the 15.16 ms the chain prices into its own weights (`PRIVATE_BATCH_ZK_VERIFY_REF_TIME_PS`, a reference hardware measurement). On reference hardware the same flood needs only about 593 ready candidates to eat the entire 9 second budget, about a third of what the default pool holds, and sustaining it costs about 8 MB per second.

## Fix verification: same flood, same everything

Running the identical attack against a producer built with the fix, a pool wrapper in `node/src/authorship.rs` that stops attempting unsigned candidates after 64 rejections in one build:

* Doomed verifications dropped to **7169**.
* The log shows exactly **112 cap warnings**, and 112 x 64 accounts for every single one of those verifications.
* Builds went back to a **median of 179 ms and a max of 357 ms**, canary latency returned to 21 s.

Perhaps this is the shape of the recommended fix, bound unsigned verification attempts per block on the producer side. Alternatives are charging weight per failed unsigned attempt, or verifying at pool admission, which `docs/BENCHMARKING_PLAN.md` already lists as a deferred option.
