# 89250 bc high missing quantum alignment in the wormhole deposit chokepoint leads to permanent freezing of funds

**Submitted on Aug 18th 2026 at 08:38:21 UTC by @FaLLenSKiLL for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89250
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The wormhole system's only deposit mechanism, an ordinary balances.transfer recorded via record\_transfer\_proof, commits ZK Merkle leaves at the raw, un-quantized deposit amount, but a spend proof can only ever attest to floor(amount / 10^10). Any deposit that isn't an exact 0.01 UNIT multiple permanently loses the remainder, and on-chain storage still shows the full amount as if it were spendable, which is misleading. In production this means ordinary users lose a fraction of their principal on essentially any deposit whose amount wasn't hand-rounded (invoice totals, fee-adjusted transfers, percentage splits), with zero attacker action required, and the same unguarded chokepoint independently strands value in three further places: mining rewards, delayed reversible-transfer payouts, and the chain's own genesis initialization.

## Vulnerability Details

hash\_leaf in pallets/zk-tree/src/tree.rs (lines 147-149) builds the leaf commitment from `amount_quantized = u32::try_from(amount / 10^10)`. That's all the exit circuit ever gets to see. Storage keeps the full raw amount in ZkLeaf.amount (pallets/zk-tree/src/lib.rs), but that field is not what a ZK proof can spend, so anyone reading it directly, a wallet or an indexer, ends up overstating what's actually recoverable.

record\_transfer\_proof at pallets/wormhole/src/lib.rs:1341 is, in the code's own words, "the single chokepoint every event-scan / call-site recorder goes through." It takes the amount it's given and passes it straight down into record\_transfer and then hash\_leaf, unchanged.

What does the job is `WormholeProofRecorderExtension:: record_proofs_from_events_since` (runtime/src/transaction\_extensions.rs: 309), a transaction extension that scans every block for Transfer, Minted, TransferOnHold and ReserveRepatriated events. That covers plain transfers, utility.batch, multisig.execute, recovery. as\_recovered, all of it, and calls record\_transfer\_proof with whatever raw amount the event carried, for whatever address it went to.

I recorded deposit of 100.005 UNIT (100 \* UNIT + SCALE/2) to TEST\_ADDRESS, a genuine wormhole address with a known secret, then computed hash\_leaf again for a second, smaller deposit (100 \* UNIT + 1) that happens to floor to the same value. The two deposits are literally indistinguishable to anything that can only see the committed hash, which is the only thing a spend proof can ever reference.

on\_finalize in the mining-rewards pallet calls mint\_reward twice per block, once for the tx-fee remint and once for the miner's share of the block reward, and both go through Currency::mint\_into plus record\_transfer\_proof with the raw amount: no rounding, no remainder handling. The fee remint in particular tends to land under one quantum by itself on quiet blocks.

do\_execute\_transfer in pallets/reversible-transfers/src/lib.rs (around line 726) runs when a delayed transfer's execution window elapses. It's a scheduler-driven hook, which the event-scan extension can't see (the same reason vesting records its own transfers explicitly), but unlike vesting, it just forwards pending.amount straight into record\_transfer\_proof. The only check anywhere in scheduling or execution is that the amount isn't zero. Any high-security account that schedules a delayed transfer of a non-round amount to a wormhole address loses the remainder the moment it executes.

Wormhole::on\_initialize, at block 1 only, walks every account in genesis storage and calls record\_transfer with each account's full balance, again with no rounding. A genesis distribution, whether validator allocations, private-sale amounts, or anything computed as a percentage or a token-sale conversion rather than hand-picked to be round, will very plausibly credit at least one wormhole address with a non-aligned amount, and that remainder is gone before a single user transaction has ever been submitted.

## Impact Details

Any deposit to any real wormhole address, made through the system's ordinary deposit mechanism, permanently loses whatever fraction of the amount falls below one 0.01-UNIT quantum. That's up to just under 0.01 UNIT per non-aligned deposit, or the entire amount if the whole deposit is smaller than one quantum. This is proven with a leaf-hash collision. Two deposits differing only below one quantum commit to the exact same leaf hash, so nothing can ever tell them apart or recover the difference.

Fee remints are stranded entirely on quiet blocks, since the real per-transaction fee is itself under one quantum, and block-reward remainders are stranded every block. The PoC measured a fee of 1e9 planck fully stranded and a reward remainder of 2449530288 planck lost out of a 692449530288 planck reward.

So does reversible-transfers. any delayed transfer for a non-aligned amount loses its remainder the moment it executes, since scheduling only checks that the amount isn't zero.

And so does genesis. every non-aligned account balance at block 1 loses its remainder before a single user transaction has run. The repo's own test helpers already exercise this code path, just with round numbers, so it's never actually been tested with realistic ones.

Whatever's already stranded stays stranded even after a fix, unless someone runs a storage migration to sweep it up.

A single transfer\_keep\_alive to a wormhole address costs 4.765 billion planck in base weight and length fee (dominated by the \~4627-byte ML-DSA-87 signature), plus WormholeProofRecorderExtension's own reserved weight for recording the resulting leaf (tree reads/writes, Poseidon hashing), modeled from the pallet's own weight constants at about 4.425 billion planck more, for a floor around 9.19 billion planck. A real transaction measured on a live node (send 100.005 UNIT from crystal\_alice to TEST\_ADDRESS, the same fixture as the PoC) paid 12.96 billion planck; the gap is most likely extra event-scan cost specific to that transaction landing in block #1, alongside genesis-leaf-recording, so both numbers are given rather than picking one.

The most you can strand in one deposit is just under ten billion planck, so the worst case returns about 1.09x what you spent using the modeled floor, or about 0.77x using the real measured fee: it costs slightly more than the maximum possible damage from that one transaction. Post-quantum signatures make a single deliberate trigger meaningfully less free than it would be on a chain with small ECDSA signatures.

That same per-transaction fee is also what gets reminted to the miner every block that has at least one signed transaction in it. Since the fee is well above one quantum by itself, a block with even one signed transaction strands the fee remint's sub-quantum portion at minimum.

Batching goes through the same event scanner. WormholeProofRecorderExtension::weight() (per\_transfer\_weight().saturating\_mul(n), runtime/src/transaction\_extensions.rs) charges each wrapped transfer its own full recording weight; only the outer extrinsic's signature and base call weight are paid once. For N=1000 wrapped transfers: cost is roughly the one-time base (4.765 billion) plus 1000 times the modeled per-transfer recording weight (4.425 billion), around 4.43 trillion planck, against damage of up to 1000 times 9,999,999,999 planck, about 10 trillion planck, a ratio around 2.3x. The per-transfer recording weight dominates over the one-time signature cost, so this ratio holds roughly flat regardless of batch size.

## References

* `https://github.com/immunefi-team/audit-comp-quantus-chain/tree/3b243b870a5442a6e5f443056f67bbba87d472f5`
* `https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1280-L1382`
* `https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L503`
* `https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L661`
* `https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/transaction_extensions.rs#L309-L390`
* `https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/zk-tree/src/tree.rs#L134-L149`
* `https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/docs/wormhole-zk.md`
* `https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L192-L237`
* `https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L726-L790`
* `https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/vesting/src/lib.rs#L561-L564`
* `https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L564-L591`

## Link to Proof of Concept

<https://gist.github.com/FaLLenSKiLL1/d9c2c7b4528599e1b0da68105ea7c89c>

## Proof of Concept

`https://gist.github.com/FaLLenSKiLL1/d9c2c7b4528599e1b0da68105ea7c89c`\
`https://gist.github.com/FaLLenSKiLL1/a96ac0b8b653bce3f6a8b401e050fe08`\
`https://gist.github.com/FaLLenSKiLL1/98ac9f1d0f1b955929daf87e77646303`

{% stepper %}
{% step %}

## Clone the repository

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain
```

Root cause, in one step: pallet\_zk\_tree::tree::hash\_leaf commits floor(amount / 1e10) into the leaf's hash. record\_transfer\_proof (pallets/wormhole/src/lib.rs) forwards whatever raw amount it receives straight into that function, with no rounding step anywhere upstream.
{% endstep %}

{% step %}

## Build and start a dev node

Instance 1, general deposit path, live node.

```bash
cargo build -p quantus-node
./target/debug/quantus-node --dev --rpc-external --rpc-cors=all --rpc-methods=unsafe --tmp
```

{% endstep %}

{% step %}

## Install the CLI and submit a non-aligned deposit

In a second terminal, install the official CLI and send a deliberately non-aligned deposit (100.005 UNIT = 100005000000000 planck, not a multiple of the 1e10-planck quantum) to TEST\_ADDRESS = qzokTZkdWXxMgSXyF86ECHxG8o8yRX5ibrX2Uw8YmqkHRdj1V:

```bash
cargo install quantus-cli --locked
quantus developer create-test-wallets
quantus send --from crystal_alice --to qzokTZkdWXxMgSXyF86ECHxG8o8yRX5ibrX2Uw8YmqkHRdj1V --amount 100.005 --wait-for-transaction
```

```
🔮 Quantus CLI
📝 Encoded call size: 42 bytes
✅ SUCCESS Transaction included. Hash: 0x0e73c5096bca6fcbba3cd236da8af4cea4680dad3e13fe47d54f8c1d77aebc96
✅ 🎉 FINISHED Transaction included in a best block.
💰 New balance: 99899.982037115 DEV
⏱️  Completed in 80.69s
```

{% endstep %}

{% step %}

## Read the recorded leaves

Once mined, check how many leaves exist and read them back:

```bash
quantus storage get --pallet ZkTree --name LeafCount --decode-as u64
```

```bash
for i in 0 1 2 3 4 5 6 7 8; do
  echo "=== leaf $i ==="
  quantus storage get --pallet ZkTree --name Leaves --key $i --key-type u64 2>&1 | grep "Raw Value"
done
```

```
=== leaf 8 ===
✅ Raw Value: 0x9fdb6c812817145c58b6eebcdc81c6c461dbe065d4d9687820938e8a120b5f0600000000000000000000000002b16fd89d0000000000000000000000
```

The returned amount field is the full raw deposit (100005000000000).
{% endstep %}

{% step %}

## Verify the live-node leaf collision

Paste wormhole\_tests.rs's second test (from the Gist) into pallets/wormhole/src/tests.rs, inside mod wormhole\_tests, then:

```bash
cargo test -p pallet-wormhole live_node_deposit_leaf -- --nocapture
```

```
running 1 test
LIVE leaf: to=be13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d (5GMvnbKw...) transfer_count=1 amount=100005000000000 exitable=100000000000000 stranded=5000000000
live_hash=[207, 102, 255, 195, 118, 226, 169, 225, 55, 113, 203, 170, 23, 243, 84, 237, 207, 144, 111, 180, 220, 205, 48, 144, 142, 111, 107, 213, 57, 32, 80, 199] smaller_remainder_hash=[207, 102, 255, 195, 118, 226, 169, 225, 55, 113, 203, 170, 23, 243, 84, 237, 207, 144, 111, 180,
220, 205, 48, 144, 142, 111, 107, 213, 57, 32, 80, 199]
test tests::wormhole_tests::live_node_deposit_leaf_matches_prediction_and_collides_with_smaller_remainder ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 72 filtered out; finished in 0.00s
```

This independently recomputes `hash_leaf` on the exact live-fetched bytes and on a hypothetical second deposit of 100\*UNIT+1 (a smaller remainder, same floor). Both recomputations produce the identical hash, proving the remainder is cryptographically absent from the leaf, not just unrounded for display.

The same live query also shows TEST\_ADDRESS's own genesis leaf (index 2, transfer\_count 0), confirming instance 4's mechanism fires on a real chain too; that genesis amount happened to be round, so nothing was stranded in that specific leaf.
{% endstep %}

{% step %}

## Verify the collision in the mock runtime

Instance 1, same collision, mock runtime (no node required). Paste wormhole\_tests.rs's first test (from the Gist) into pallets/wormhole/src/tests.rs, inside mod wormhole\_tests, then:

```bash
cargo test -p pallet-wormhole ordinary_deposit_with_non_quantum -- --nocapture
```

```
running 1 test
deposit=100005000000000 exitable=100000000000000 stranded=5000000000
test tests::wormhole_tests::ordinary_deposit_with_non_quantum_amount_strands_the_remainder_at_the_real_wa ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 72 filtered out; finished in 0.00s
```

{% endstep %}

{% step %}

## Verify mining-rewards impact

Instance 2, mining-rewards. Paste mining\_rewards\_test.rs (from the Gist) into pallets/mining-rewards/src/tests.rs, then:

```bash
cargo test -p pallet-mining-rewards quantum_leaf_floor -- --nocapture
```

```
running 1 test
fee_mint=1000000000 frozen=1000000000
miner_reward=692449530288 frozen=2449530288
credited=693449530288 total_frozen=3449530288
test tests::quantum_leaf_floor_strands_fee_mint_and_reward_remainder_on_wormhole_miner ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 28 filtered out; finished in 0.01s
```

{% endstep %}

{% step %}

## Verify reversible-transfers impact

Instance 3, reversible-transfers. Paste reversible\_transfers\_test.rs (from the Gist) into pallets/reversible-transfers/src/tests/test\_reversible\_transfers.rs, then:

```bash
cargo test -p pallet-reversible-transfers scheduled_transfer_of_a_non_quantum -- --nocapture
```

```
running 1 test
reversible-transfer executed: amount=105000000000 exitable=100000000000 stranded=5000000000
test tests::test_reversible_transfers::scheduled_transfer_of_a_non_quantum_aligned_amount_strands_the_remainder_on_execute ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 51 filtered out; finished in 0.03s
```

{% endstep %}
{% endstepper %}
