# #37500 \[BC-Low] Blocklist can be circumvented due to incorrect blocking logic in \`request\_decider::c

**Submitted on Dec 6th 2024 at 12:31:58 UTC by @n4nika for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #37500
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Note to immunefi

This issue was discussed with the Stacks team on discord and I was asked by @Evonide to submit it without the need for a PoC, which is why none is provided on this issue

## Summary

`request_decider::can_accept_deposit_request` currently returns `Ok(true)` if there is at least one non-blocklisted address as an input to a deposit UTXO. This means anyone can circumvent the blocklist by adding a tiny non-blocklisted UTXO as an input to their deposit request.

## Finding Description

`can_accept_deposit_request` uses the following code to determine whether a certain UTXO can be accepted:

```rs
async fn can_accept_deposit_request(&self, req: &model::DepositRequest) -> Result<bool, Error> {

    // [getting addresses from the request]

    let responses = futures::stream::iter(&addresses)
        .then(|address| async { client.can_accept(&address.to_string()).await })
        .inspect_err(|error| tracing::error!(%error, "blocklist client issue"))
        .collect::<Vec<_>>()
        .await;

    // If any of the inputs addresses are fine then we pass the deposit
    // request.
    let can_accept = responses.into_iter().any(|res| res.unwrap_or(false));
    Ok(can_accept)
}
```

As we can see, we create a `responses` vector with elements for each input address, either being `true` if the address can be accepted or `false` if it should be rejected.

Then `can_accept` is determined by using `.any` on the responses. `any` returns `true` if there is at least one `true` element in `responses`.

## Impact

Say a user has a big UTXO which is blacklisted but wants to deposit it. They can just create a deposit request including a very small UTXO from a non-blocklisted address, allowing them to successfully deposit their bitcoins to Stacks from an address which should not be allowed to do so.

## Mitigation

Consider using `.all` instead of `.any` to ensure none of the inputs to the UTXO come from blocklisted addresses.

## Proof of Concept

Omitted due to direct request from Stacks team
