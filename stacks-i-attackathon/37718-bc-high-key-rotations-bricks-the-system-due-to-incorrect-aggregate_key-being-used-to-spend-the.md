# #37718 \[BC-High] Key rotations bricks the system due to incorrect \`aggregate\_key\` being used to spen

**Submitted on Dec 13th 2024 at 08:31:21 UTC by @n4nika for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #37718
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Summary

When the coordinator gets the signatures to spend the previous `peg-UTXO`, they need to pass the `aggregate_key` which matches the key of that UTXO's key-only spendable taproot output. Currently this is not the case if we rotated the keys in the current signing round. This causes the coordinator to get a signature for the sweep transaction which cannot actually spend it. Since there is no programmed fallback to fix this, this will completely shutdown the system.

## Finding Description

In the coordinator we have the following flow:

1. get requests from `emily`
2. deploy smart contracts if needed
3. handle key rotation
4. get pre-signature (just pre-validation)
5. handle `sweep` transactions
6. handle sBTC deposit transactions (`s-deposit`) (these mint sBTC)
   1. get signatures from signers
   2. submit the TX to Stacks

If we look at the code, we see that the newly gotten `aggregate_key` is used if we just rotated the keys:

```rs
let aggregate_key = match maybe_aggregate_key {
    Some(key) => key,
    // This function returns the new DKG aggregate key.
    None => {
        let dkg_result = self.coordinate_dkg(&bitcoin_chain_tip).await?;
        self.get_signer_set_and_aggregate_key(&bitcoin_chain_tip)
            .await
            .ok()
            .and_then(|res| res.0)
            .unwrap_or(dkg_result)
    }
};

self.deploy_smart_contracts(&bitcoin_chain_tip, &aggregate_key)
    .await?;

self.check_and_submit_rotate_key_transaction(&bitcoin_chain_tip, &aggregate_key)
    .await?;

self.construct_and_sign_bitcoin_sbtc_transactions(
    &bitcoin_chain_tip,
    &aggregate_key, // [1] <-----
    &signer_public_keys,
)
.await?;
```

At `[1]`, we pass it to the function handling the sweep transactions (which includes handling of the `peg UTXO`).

That function then calls `sign_and_broadcast` which then coordinates signing rounds in order to spend the `peg UTXO` and all user deposits. What is important here, is the signing of the `peg UTXO`:

```rs
let mut coordinator_state_machine = CoordinatorStateMachine::load(
    &mut self.context.get_storage_mut(),
    *aggregate_key, // [2] <-----
    signer_public_keys.clone(),
    self.threshold,
    self.private_key,
)
.await?;
// [...]
let signature = self
    .coordinate_signing_round(
        bitcoin_chain_tip,
        &mut coordinator_state_machine, // [3] <-----
        txid,
        &msg,
        SignatureType::Taproot(None),
    )
    .await;
```

At `[2]` we create the state machine used for signing with the `aggregate_key` and request signatures using that state machine at `[3]`.

### Scenario

This becomes an issue if we rotate the keys:

* we have an `aggregate_key` (`keyA`) which we use to sign `peg UTXOs`. This means all our newly generated UTXOs can only be spent with `keyA`!
* we now rotate our keys, generating a new key (`keyB`)
* since we use `keyB` in `sign_and_broadcast` now, the resulting signature will NOT be able to spend the previous `UTXO` since it is only spendable with `keyA`!

## Impact

This will cause a full system halt since we cannot execute new sweep transactions and our previous `peg UTXO` is locked until we execute a signer upgrade.

## Mitigation

In order to fix this, I would recommend getting the needed `aggregate_key` from the previous `peg UTXO` itself. This should work since signers are still able to sign for old `aggregate_key`s even after a rotation (according to the team).

## Proof of Concept

## Note to immunefi

This issue was discussed in discord and I was asked by @djordon to submit it explicitly without the requirement to write a PoC
