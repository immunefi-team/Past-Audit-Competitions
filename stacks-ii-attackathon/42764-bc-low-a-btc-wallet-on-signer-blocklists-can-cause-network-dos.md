# #42764 \[BC-Low] A BTC wallet on signer blocklists can cause network DoS

**Submitted on Mar 26th 2025 at 03:47:19 UTC by @Blobism for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #42764
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * Temporarily Freezing Network Transactions

## Description

## Brief/Intro

The current transaction packaging algorithm is flawed in that it prioritizes transactions which have signer votes against them. This means that the owner of a BTC wallet that is present on some blocklists can submit a large set of deposit requests and get priority over all other deposit/withdrawal requests for the current bitcoin block tenure, leading to network DoS.

## Vulnerability Details

The main issue is function `compute_optimal_packages` in `signer/src/bitcoin/packaging.rs`:

```rust
// This is a variant of the Best-Fit-Decreasing algorithm, so we sort
// by "weight" decreasing. We use the votes against as the weight, but
// vsize is a reasonable weight metric as well.
let mut item_vec: Vec<(u32, T)> = items
    .into_iter()
    .map(|item| (item.votes().count_ones(), item))
    .collect();

item_vec.sort_by_key(|(vote_count, _)| std::cmp::Reverse(*vote_count));
```

Here, votes refers to votes **against** a particular item. This means that any items with votes against them will get priority over items without votes. A malicious user with a BTC wallet address in signer blocklists can exploit this to get priority in signing rounds where those signers are present. As long as the number of votes against them does not exceed the threshold, their requests are valid and get priority.

The actual attack comes from the fact that the maximum number of deposits in a package is 625 (25 final txs per package \* 25 deposits per final tx). This is sufficiently low such that a malicious user could submit this many deposit requests for each new bitcoin block, temporarily preventing all other transactions.

## Impact Details

The impact is that an attacker can temporarily freeze network transactions. The attack is made possible when a BTC wallet of the attacker has votes against it from some of the signers, but not an excessive number. The attack could be improved by vote manipulation if a malicious signer is involved as well.

It would be somewhat impractical and expensive to keep repeating this process of DoSing the system, since it requires 625 deposit requests per bitcoin block, which could cost > $100 USD in transaction fees. Therefore, I classify this as a temporary network DoS.

## References

Branch: <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>

Commit on branch: <https://github.com/stacks-network/sbtc/commit/79e0caf06f079cee08831fdc13d21de5459170b9>

## Link to Proof of Concept

<https://gist.github.com/blobism/a56ff0cce0773b85644ee024b8dfa70d>

## Proof of Concept

Add the unit test in the Gist to `signer/src/bitcoin/utxo.rs`: <https://gist.github.com/blobism/a56ff0cce0773b85644ee024b8dfa70d>

Run it like so:

```bash
cargo test --package signer --lib -- bitcoin::utxo::tests::deposits_from_btc_wallet_on_blocklists_exploit --exact --show-output
```

The unit test demonstrates a case where there are 625 deposit requests from the malicious user, and 1 other deposit request that they are trying to censor. The malicious user deposit requests all have 1 vote against them, while the other deposit request has no votes against it.

The unit test will show that the resulting package includes all of the malicious user deposit requests, while the other deposit request is left out of the package.
